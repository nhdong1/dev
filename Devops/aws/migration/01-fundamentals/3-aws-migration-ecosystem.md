# AWS Migration Ecosystem — Tổng Quan Hệ Sinh Thái

> AWS cung cấp một bộ công cụ và dịch vụ toàn diện phục vụ mọi giai đoạn migration. Hiểu rõ "công cụ nào dùng cho việc gì" là kỹ năng thiết yếu cho Migration Engineer và Solutions Architect.

## 📚 Mục Lục

1. [Bản Đồ Dịch Vụ Migration](#bản-đồ-dịch-vụ-migration)
2. [Nhóm 1 — Lập Kế Hoạch & Đánh Giá](#nhóm-1--lập-kế-hoạch--đánh-giá)
3. [Nhóm 2 — Di Chuyển Ứng Dụng](#nhóm-2--di-chuyển-ứng-dụng)
4. [Nhóm 3 — Di Chuyển Cơ Sở Dữ Liệu](#nhóm-3--di-chuyển-cơ-sở-dữ-liệu)
5. [Nhóm 4 — Truyền Tải Dữ Liệu Qua Mạng](#nhóm-4--truyền-tải-dữ-liệu-qua-mạng)
6. [Nhóm 5 — Truyền Tải Dữ Liệu Ngoại Tuyến](#nhóm-5--truyền-tải-dữ-liệu-ngoại-tuyến)
7. [Nhóm 6 — Hiện Đại Hóa](#nhóm-6--hiện-đại-hóa)
8. [Ma Trận Chọn Dịch Vụ](#ma-trận-chọn-dịch-vụ)
9. [Decision Trees Theo Use Case](#decision-trees-theo-use-case)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Bản Đồ Dịch Vụ Migration

```
AWS MIGRATION ECOSYSTEM
════════════════════════════════════════════════════════════════

PHASE: ASSESS (Đánh Giá)
├── Application Discovery Service (ADS) — Khám phá hạ tầng
├── Migration Evaluator — Phân tích TCO, đề xuất
└── Migration Hub — Theo dõi tiến trình tập trung

PHASE: MIGRATE (Di Chuyển)
│
├── DI CHUYỂN MÁY CHỦ / ỨNG DỤNG:
│   ├── MGN (Application Migration Service) — Rehost servers
│   └── Elastic Disaster Recovery (DRS) — DR liên tục
│
├── DI CHUYỂN CƠ SỞ DỮ LIỆU:
│   ├── DMS (Database Migration Service) — Di chuyển data
│   └── SCT (Schema Conversion Tool) — Chuyển đổi schema
│
├── TRUYỀN TẢI DỮ LIỆU QUA MẠNG:
│   ├── DataSync — Đồng bộ file/object tốc độ cao
│   ├── Transfer Family — SFTP/FTPS/FTP managed
│   └── Storage Gateway — Kết nối on-premises với S3/EBS/Tape
│
└── TRUYỀN TẢI DỮ LIỆU NGOẠI TUYẾN (OFFLINE):
    ├── Snowcone (8-14 TB) — Thiết bị nhỏ, vùng xa
    ├── Snowball Edge (80-210 TB) — Di chuyển lớn, có tính toán
    └── Snowmobile (100 PB) — Xe tải dữ liệu exabyte

PHASE: MODERNIZE (Hiện Đại Hóa)
├── Mainframe Modernization — Refactor mainframe lên microservices
├── App2Container — Container hóa Java/.NET tự động
└── End-of-Support Migration Program (EMP) — Windows/SQL Server EOS
```

---

## 📊 Nhóm 1 — Lập Kế Hoạch & Đánh Giá

### AWS Application Discovery Service (ADS)

> **ADS** — Tự động khám phá và thu thập thông tin về hạ tầng on-premises để lập kế hoạch migration.

#### Hai Chế Độ Hoạt Động

```
AGENTLESS DISCOVERY (Không Cài Agent):
├── Công cụ: AWS Agentless Discovery Connector (VM appliance OVA)
├── Triển khai: Import vào VMware vSphere, kết nối vCenter
├── Thu thập:
│   ├── VM inventory (hostname, OS, CPU, RAM, disk)
│   ├── Resource utilization (avg/peak)
│   └── Network connections cơ bản
├── Ưu điểm: Deploy nhanh, không cần agent từng server
├── Nhược điểm: Ít chi tiết hơn, chỉ VMware
└── Phù hợp: Môi trường VMware, cần kết quả nhanh

AGENT-BASED DISCOVERY (Cài Agent):
├── Công cụ: AWS Discovery Agent (cài trên từng server)
├── Hỗ trợ: Windows, Linux (vật lý và ảo)
├── Thu thập:
│   ├── Tất cả như agentless +
│   ├── Danh sách process đang chạy
│   ├── Network connections chi tiết (IP:port)
│   └── Performance metrics theo thời gian
├── Ưu điểm: Dependency mapping chính xác
├── Nhược điểm: Phải cài từng server, tốn thời gian
└── Phù hợp: Cần biết chính xác ứng dụng nào kết nối ứng dụng nào
```

#### Đầu Ra và Tích Hợp

```
Dữ liệu ADS → AWS Migration Hub:
├── Xem dependency maps trực quan
├── Group servers thành applications
├── Export CSV để phân tích với Migration Evaluator
└── Import vào MGN/DMS để theo dõi migration status
```

---

### AWS Migration Evaluator (Trước Đây: TSO Logic)

> **Migration Evaluator** — Phân tích dữ liệu on-premises để xây dựng **business case** với số liệu TCO cụ thể và đề xuất right-sizing.

#### Cách Hoạt Động

```
Input:
├── Data từ ADS (discovery data)
├── Hoặc manual import CSV (từ RVTools, SCCM, VMware vRealize)
└── Thông tin license on-premises

Phân Tích:
├── So sánh chi phí on-premises vs AWS
├── Đề xuất EC2 instance types phù hợp (right-sizing)
├── Tính toán savings từ Reserved Instances
└── Tính impact của BYOL — Bring Your Own License

Output: Báo cáo Migration Business Case:
├── Estimated 3-year TCO on-premises vs AWS
├── Potential savings %
├── Recommended EC2 sizes theo workload
└── Quick insights cho executive presentation
```

#### Khác Biệt Migration Evaluator vs AWS Pricing Calculator

| | Migration Evaluator | AWS Pricing Calculator |
| - | ------------------- | ---------------------- |
| **Mục đích** | TCO business case | Ước tính chi phí AWS |
| **Input** | Data thực tế từ on-premises | Cấu hình tự nhập |
| **Output** | So sánh on-prem vs AWS | Chỉ chi phí AWS |
| **Khi dùng** | Pre-migration planning | Thiết kế solution mới |

---

### AWS Migration Hub

> **Migration Hub** — Trung tâm theo dõi **tiến trình migration** tập trung, tích hợp với các công cụ migration khác.

#### Tính Năng Chính

```
Tính năng:
├── Dashboard tổng quan: % servers migrated
├── Per-server tracking: Not Started / In Progress / Migrated
├── Tích hợp native: MGN, DMS
├── Tích hợp partner tools: CloudEndure, Carbonite, ATADATA
├── Migration Hub Orchestrator: Tự động hóa workflow migration
└── Miễn phí sử dụng (chỉ trả tiền cho dịch vụ dùng bên dưới)

Không gian lưu trữ (Home Region):
├── Phải chọn Home Region cho Migration Hub
├── Tất cả tracking data lưu ở đây
└── Không thể thay đổi sau khi đã chọn
```

---

## 🖥️ Nhóm 2 — Di Chuyển Ứng Dụng

### AWS MGN — Application Migration Service

> **AWS MGN** (Application Migration Service) — Dịch vụ **Rehost (lift-and-shift)** tự động, di chuyển server vật lý, ảo, hoặc cloud khác sang EC2.

#### Kiến Trúc Hoạt Động

```
SOURCE SERVER (On-Premises)
      │
      │ Cài AWS Replication Agent
      ▼
REPLICATION AGENT
      │ Liên tục replicate block-level (không interrupt production)
      ▼
AWS STAGING AREA (trong AWS account của bạn):
├── Lightweight staging EC2 instance
├── EBS volumes (mirror của source disk)
└── Replication Server (quản lý data flow)
      │
      │ Khi ready để test/cutover
      ▼
LAUNCH TEMPLATE (cấu hình EC2 đích):
├── Instance type, subnet, security groups
├── Instance tags, IAM role
└── Post-launch scripts (nếu cần)
      │
      ▼
TEST CUTOVER ──────────► Target EC2 Instance (test)
      │                   └── Chạy smoke tests, không ảnh hưởng production
      │
      ▼ (Sau khi test pass)
PRODUCTION CUTOVER ──────► Target EC2 Instance (production)
      │                     └── Thay thế source server
      ▼
FINALIZE:
├── Xóa staging resources
├── Terminate source server (nếu muốn)
└── Mark as Migrated trong Migration Hub
```

#### Hỗ Trợ Platform

```
Source (Nguồn) được hỗ trợ:
├── Physical servers (máy chủ vật lý)
├── VMware vSphere VMs
├── Microsoft Hyper-V VMs
├── Các cloud khác: Azure, GCP, Alibaba Cloud
└── OS: Windows Server 2003+, nhiều distro Linux

Target (Đích):
└── Amazon EC2 instance (bất kỳ loại nào)
```

#### MGN vs Server Migration Service (SMS)

```
AWS SMS (cũ, deprecated 2022):
├── Chỉ VMware, Hyper-V, Azure
└── Agent-based, phức tạp hơn

AWS MGN (mới, thay thế SMS):
├── Hỗ trợ nhiều platform hơn
├── Replication liên tục, không cần maintenance window dài
├── Test cutover không ảnh hưởng production
└── Đây là dịch vụ được khuyến nghị hiện tại
```

---

### AWS Elastic Disaster Recovery (DRS)

> **AWS DRS** — Cung cấp **disaster recovery (phục hồi thảm họa)** liên tục bằng cách replicate workload on-premises sang AWS, sẵn sàng failover trong phút.

```
DRS vs MGN:
├── DRS: Mục đích DR (Disaster Recovery — Phục Hồi Thảm Họa)
│   └── Luôn chạy replication, failover khi disaster xảy ra
└── MGN: Mục đích Migration (di chuyển một lần, sau đó kết thúc)

DRS Use Case:
├── Bảo vệ on-premises workload với RTO vài phút, RPO vài giây
├── Kết hợp Migration + DR: Dùng MGN để migrate, sau đó DRS để DR
└── Multi-region DR (AWS → AWS)
```

---

## 🗄️ Nhóm 3 — Di Chuyển Cơ Sở Dữ Liệu

### AWS DMS — Database Migration Service

> **AWS DMS** — Dịch vụ di chuyển cơ sở dữ liệu, hỗ trợ cả **homogeneous migration (di chuyển đồng nhất)** (MySQL → MySQL) và **heterogeneous migration (di chuyển dị cấu trúc)** (Oracle → Aurora).

#### Kiến Trúc DMS

```
SOURCE DATABASE ──────────────────────────────► TARGET DATABASE
(On-premises / EC2 / RDS)                       (RDS / Aurora / Redshift)
        │                                               ▲
        │                                               │
        └──────────► REPLICATION INSTANCE ─────────────┘
                     (EC2 t3.medium, r5.2xlarge...)

Replication Instance:
├── EC2 instance quản lý data flow
├── Kích thước ảnh hưởng throughput
└── Multi-AZ option cho high availability

Tasks:
├── Full Load: Di chuyển dữ liệu hiện có một lần
├── CDC Only: Chỉ capture thay đổi (Change Data Capture)
└── Full Load + CDC: Di chuyển toàn bộ, sau đó sync liên tục
     └── Đây là phương pháp phổ biến nhất cho zero-downtime
```

#### CDC — Change Data Capture (Capture Thay Đổi Liên Tục)

```
CDC cho phép migration zero-downtime:

1. Full Load: Copy toàn bộ data từ source → target (vài giờ/ngày)
2. CDC: Liên tục sync INSERT/UPDATE/DELETE trong khi source vẫn chạy
3. Lag giảm về gần 0 (< 30 giây)
4. Cutover: Stop writes trên source, chờ CDC sync xong, đổi connection string
```

#### Databases Được Hỗ Trợ

```
Source (nguồn):
├── Oracle, SQL Server, MySQL, PostgreSQL, MariaDB
├── MongoDB, IBM DB2, Sybase
└── SAP ASE, Azure SQL Database, Google Cloud SQL

Target (đích):
├── Amazon RDS (tất cả engines)
├── Amazon Aurora (MySQL, PostgreSQL)
├── Amazon Redshift (data warehouse)
├── Amazon DynamoDB
├── Amazon S3
└── Amazon OpenSearch Service
```

---

### AWS SCT — Schema Conversion Tool

> **AWS SCT** — Công cụ tự động **chuyển đổi schema (cấu trúc cơ sở dữ liệu)** và code (stored procedures, triggers, views) giữa các database engine khác nhau.

```
SCT cần thiết khi:
└── Heterogeneous migration (Oracle → Aurora PostgreSQL, SQL Server → MySQL)

Không cần SCT khi:
└── Homogeneous migration (MySQL → RDS MySQL, PostgreSQL → Aurora PostgreSQL)

SCT tự động chuyển đổi:
├── Tables, views, indexes (thường 80-95% tự động)
├── Stored procedures, functions (phức tạp hơn, có thể cần sửa tay)
├── Triggers, sequences
└── Assessment report: % code tự động convert được, % cần sửa tay

SCT không làm được:
├── Chuyển đổi 100% mọi trường hợp (không có tool nào làm được)
├── Semantic testing (kiểm tra logic nghiệp vụ đúng không)
└── Performance tuning sau conversion
```

---

## 📡 Nhóm 4 — Truyền Tải Dữ Liệu Qua Mạng

### AWS DataSync

> **AWS DataSync** — Dịch vụ **đồng bộ dữ liệu** tự động, nhanh, và an toàn giữa storage on-premises và AWS storage services.

#### Kiến Trúc DataSync

```
SOURCE LOCATION                    TARGET LOCATION
(On-Premises)                       (AWS)
├── NFS server          ┌──────────► Amazon S3
├── SMB server          │           ├── Amazon EFS
├── HDFS                │           └── Amazon FSx
├── Object storage      │
└── S3-compatible    DATASYNC AGENT
                     (VM appliance cài on-premises)
                         │
                         ▼
                    DataSync SERVICE (AWS managed)
                         │
                         └──► Task: Schedule, filter, verify
```

#### Tính Năng Nổi Bật

```
Tốc độ: Tới 10 Gbps per agent (multi-threaded, parallel)
Toàn vẹn dữ liệu: Tự động checksum verification sau transfer
Mã hóa: TLS encryption in-transit
Lọc: Include/exclude patterns (ví dụ: chỉ sync *.log)
Lập lịch: Chạy theo schedule hoặc on-demand
Bandwidth throttling: Giới hạn băng thông để không ảnh hưởng business traffic
```

#### DataSync vs aws s3 cp / rsync

| | DataSync | rsync / aws s3 cp |
| - | -------- | ----------------- |
| **Tốc độ** | Tối ưu, multi-threaded | Phụ thuộc bandwidth thực tế |
| **Verification** | Tự động checksum | Cần tự làm |
| **Scheduling** | Built-in | Cần script + cron |
| **Monitoring** | CloudWatch tích hợp | Tự làm |
| **Chi phí** | Theo GB transferred | Chỉ chi phí bandwidth |
| **Phù hợp** | Migration lớn, production | Test, migration nhỏ |

---

### AWS Transfer Family

> **AWS Transfer Family** — Dịch vụ **managed SFTP/FTPS/FTP/AS2 (giao thức truyền file)** trực tiếp vào Amazon S3 hoặc Amazon EFS, không cần quản lý server.

```
Giao thức hỗ trợ:
├── SFTP (SSH File Transfer Protocol) — Phổ biến nhất
├── FTPS (FTP over TLS) — Legacy enterprise
├── FTP (không mã hóa) — Chỉ dùng nội bộ
└── AS2 (Applicability Statement 2) — EDI/B2B business exchange

Use case phổ biến:
├── Thay thế on-premises SFTP server
├── B2B file exchange (đối tác gửi file vào S3 qua SFTP)
├── Data ingestion pipeline từ đối tác bên ngoài
└── Compliance: audit logs, encryption tự động

Lợi ích so với tự chạy SFTP server:
├── Không quản lý EC2, patching, HA
├── Scale tự động
├── Tích hợp IAM để kiểm soát quyền
└── Ghi log vào CloudWatch tự động
```

---

### AWS Storage Gateway

> **AWS Storage Gateway** — Kết nối on-premises applications với AWS cloud storage, giúp on-premises apps đọc/ghi data trên AWS như storage local.

```
Ba Loại Gateway:

1. FILE GATEWAY (Cổng File):
   On-premises NFS/SMB client ←→ File Gateway ←→ Amazon S3
   Use case: Backup files on-premises lên S3, archive

2. VOLUME GATEWAY (Cổng Volume):
   On-premises iSCSI storage ←→ Volume Gateway ←→ Amazon S3/EBS
   ├── Cached mode: Data chính trên S3, cache locally
   └── Stored mode: Data chính on-premises, backup lên S3

3. TAPE GATEWAY (Cổng Băng Từ):
   On-premises backup app ←→ Tape Gateway ←→ Amazon S3/Glacier
   Use case: Thay thế vật lý tape backup, virtual tape library (VTL)
```

---

## 📦 Nhóm 5 — Truyền Tải Dữ Liệu Ngoại Tuyến

> Khi băng thông mạng không đủ hoặc khi transfer qua internet tốn quá nhiều thời gian, AWS cung cấp Snow Family — thiết bị vật lý để transfer data ngoại tuyến (offline).

### Quy Tắc Vàng Chọn Snow

```
Thời gian transfer qua mạng > 1 tuần → Dùng Snow Family
(Ví dụ: 10 TB qua đường 10 Mbps = 10 TB / 10 Mbps ≈ 100 ngày → Dùng Snow)
```

### AWS Snowcone

```
Thông số:
├── Dung lượng: 8 TB HDD hoặc 14 TB SSD
├── Kích thước: Nhỏ bằng hộp giày (2.1 kg)
├── Compute: 2 vCPU, 4 GB RAM (chạy EC2/Lambda tại edge)
└── Kết nối: WiFi, USB-C, PoE

Use cases:
├── Vùng xa xôi, không có data center (công trường, tàu, rừng)
├── Di chuyển dữ liệu nhỏ
└── Edge computing + data collection

Đặc biệt:
└── Có thể dùng DataSync tích hợp để transfer về AWS qua mạng
    (khi có internet, không cần gửi thiết bị)
```

### AWS Snowball Edge

```
Hai phiên bản:

STORAGE OPTIMIZED (Tối Ưu Lưu Trữ):
├── 80 TB usable storage
├── 40 vCPU, 80 GB RAM
└── Phù hợp: Di chuyển dữ liệu lớn là chủ yếu

COMPUTE OPTIMIZED (Tối Ưu Tính Toán):
├── 28 TB usable storage (hoặc 42 TB HDD)
├── 52 vCPU, 208 GB RAM
├── GPU option (NVIDIA V100)
└── Phù hợp: Edge computing, ML inference, video processing

Tính năng chung:
├── Chạy EC2 instances và Lambda functions at the edge
├── Mã hóa AES-256 bit
├── Trusted Platform Module (TPM) — Chip bảo mật phần cứng
└── Có thể cluster 15 thiết bị để có petabyte-scale
```

### AWS Snowmobile

```
Snowmobile là gì:
├── Xe tải chứa container data center (exabyte-scale)
├── Dung lượng: 100 PB (petabyte) mỗi xe
├── Đến tận data center của bạn, kết nối vật lý
└── Bảo mật: Vệ sĩ, camera, GPS tracking, mã hóa

Khi nào dùng:
├── > 10 PB dữ liệu cần di chuyển
└── Ví dụ: Video library lớn, sensor data archive, genomics

Thực tế:
└── Rất ít use case dùng Snowmobile (chỉ tổ chức cực lớn)
    Nhưng thường xuất hiện trong câu hỏi AWS exam
```

### So Sánh Snow Family

| | Snowcone | Snowball Edge Storage | Snowball Edge Compute | Snowmobile |
| - | -------- | --------------------- | --------------------- | ---------- |
| **Dung lượng** | 8-14 TB | 80 TB | 28-42 TB | 100 PB |
| **Compute** | 2 vCPU | 40 vCPU | 52 vCPU | N/A |
| **Kích thước** | Hộp giày | Vali | Vali | Xe tải |
| **Use case** | Vùng xa, nhỏ | Migration lớn | Edge computing | Exabyte |
| **Số lượng** | 1 | 1-15 | 1 | 1 |

---

## 🔄 Nhóm 6 — Hiện Đại Hóa

### AWS Mainframe Modernization

```
Vấn đề: Nhiều enterprise vẫn chạy COBOL/PL1 trên mainframe IBM z/OS
Chi phí mainframe: Cực kỳ cao (hardware + software + nhân sự đặc biệt)

AWS Mainframe Modernization cung cấp:

1. REFACTOR (Tái Cấu Trúc):
   COBOL apps → Micro Focus (tự động convert sang Java/C#)
   └── Chạy trên ECS/EKS, không cần mainframe

2. REPLATFORM (Đổi Nền Tảng):
   COBOL apps → AWS Blu Age (automated refactoring)
   └── Chạy native Java trên AWS

Dịch vụ tích hợp:
├── AWS Blu Insights: Phân tích code COBOL
├── AWS Blu Age: Tự động chuyển COBOL → Java
└── AWS Micro Focus: Emulate mainframe runtime trên EC2
```

### AWS App2Container (A2C)

> **App2Container** — Tự động **container hóa (containerize)** ứng dụng Java và .NET đang chạy, tạo Dockerfile và Kubernetes/ECS manifests.

```
Quy trình A2C:
1. Phân tích ứng dụng đang chạy (analyze)
2. Tạo Docker image
3. Tạo ECS Task Definition hoặc Kubernetes deployment manifest
4. Push image lên Amazon ECR

Phù hợp:
├── Java (Spring Boot, Tomcat) trên EC2
└── .NET (ASP.NET) trên Windows Server

Giới hạn:
├── Không xử lý được dependencies phức tạp
└── Cần kiểm tra thủ công sau khi containerize
```

---

## 📊 Ma Trận Chọn Dịch Vụ

### Chọn Dịch Vụ Migration Máy Chủ

```
Muốn di chuyển server / ứng dụng?
├── Lift-and-shift (không thay đổi) → AWS MGN
├── Di chuyển VMware VM giữ nguyên VMware → VMware Cloud on AWS + HCX
└── Disaster Recovery (bảo vệ on-premises) → AWS DRS
```

### Chọn Dịch Vụ Migration Database

```
Muốn di chuyển database?
├── Cùng engine (MySQL → RDS MySQL) → DMS (không cần SCT)
├── Khác engine (Oracle → Aurora) → SCT để chuyển schema + DMS để migrate data
├── Zero-downtime → DMS với Full Load + CDC
└── Chỉ cần đồng bộ liên tục → DMS CDC task
```

### Chọn Dịch Vụ Truyền Tải File

```
Muốn di chuyển files / objects?
│
├── Có băng thông tốt, file/object storage?
│   ├── Di chuyển một lần hoặc đồng bộ liên tục → DataSync
│   ├── Đối tác cần upload via SFTP/FTP → Transfer Family
│   └── On-prem apps cần dùng S3 như network drive → Storage Gateway (File Gateway)
│
└── Băng thông kém hoặc > 10 TB?
    ├── < 14 TB, vùng xa → Snowcone
    ├── 14 TB - 80 TB → Snowball Edge Storage
    ├── Cần edge computing + storage → Snowball Edge Compute
    └── > 10 PB → Snowmobile
```

---

## 🌳 Decision Trees Theo Use Case

### Use Case: Di Chuyển 500 Servers

```
Phân loại 500 servers:

1. Servers chạy VMware, muốn giữ VMware nguyên?
   → Relocate: VMware Cloud on AWS

2. Servers cần di chuyển lên EC2 nhanh?
   → Rehost: Cài AWS Replication Agent → Dùng AWS MGN

3. Servers chạy Windows Server 2003 (không còn support)?
   → End-of-Support Migration Program (EMP)
```

### Use Case: Thay Thế SFTP Server

```
Hiện có SFTP server on-premises, muốn move lên AWS?

Câu hỏi: Có cần giữ giao thức SFTP không?
├── Có → AWS Transfer Family (SFTP endpoint, data vào S3)
└── Không → S3 + presigned URLs hoặc S3 + API
```

### Use Case: Backup File Servers Lên Cloud

```
Có NFS/SMB file server on-premises, muốn backup lên AWS?

Câu hỏi: Loại backup như thế nào?
├── Archive files lên S3 → DataSync (one-time hoặc scheduled)
├── On-premises apps đọc/ghi S3 như local drive → Storage Gateway File Gateway
└── Backup tape vào cloud → Storage Gateway Tape Gateway
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Phân biệt AWS MGN và AWS DMS?**

> A:
> - **AWS MGN** (Application Migration Service): Di chuyển **toàn bộ server** (OS + ứng dụng + data) lên EC2. Dùng cho Rehost strategy. Không quan tâm đến loại database hay ứng dụng gì.
> - **AWS DMS** (Database Migration Service): Chỉ di chuyển **dữ liệu trong database**. Cần cấu hình source endpoint, target endpoint, và replication task. Hỗ trợ nhiều database engines.
>
> Thường dùng cùng nhau: MGN migrate app servers, DMS migrate databases.

**Q: Khi nào dùng DataSync thay vì Snow Family?**

> A: Quy tắc thực tế:
> - **DataSync**: Khi có băng thông tốt và transfer time < 1 tuần
> - **Snow Family**: Khi data > 10 TB và/hoặc transfer time qua mạng > 1 tuần
>
> Ví dụ: 50 TB qua đường 1 Gbps = 50 TB / 1 Gbps ≈ 4.6 ngày → DataSync
> Ví dụ: 50 TB qua đường 100 Mbps = 50 TB / 100 Mbps ≈ 46 ngày → Snowball Edge

**Q: Transfer Family khác gì so với tự dựng SFTP server trên EC2?**

> A: Transfer Family là **managed service** — AWS quản lý infrastructure:
> - Không cần patch, upgrade, quản lý EC2 OS
> - Auto-scaling tự động theo lượng kết nối
> - High availability built-in (không cần setup Multi-AZ tự tay)
> - Tích hợp sẵn với S3/EFS (không cần mount volume)
> - Logging tự động vào CloudWatch
>
> Trade-off: Ít tùy biến hơn, chi phí cao hơn EC2 cho traffic lớn.

**Q: Storage Gateway có mấy loại và khi nào dùng?**

> A: Ba loại:
> 1. **File Gateway**: On-premises NFS/SMB → S3. Dùng khi app cần đọc/ghi file như mạng chia sẻ nhưng lưu trên S3.
> 2. **Volume Gateway**: On-premises iSCSI block storage → S3/EBS. Dùng cho backup block volumes on-premises lên cloud.
> 3. **Tape Gateway**: Thay thế physical tape backup bằng virtual tape library, lưu trên S3 Glacier. Dùng khi đang dùng backup software hỗ trợ tape (Veeam, Backup Exec).

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
