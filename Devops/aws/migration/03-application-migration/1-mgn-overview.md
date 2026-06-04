# AWS MGN — Application Migration Service: Tổng Quan Và Kiến Trúc

> **AWS MGN** — Application Migration Service (Dịch Vụ Di Chuyển Ứng Dụng) — là dịch vụ thế hệ mới của AWS để rehost (nâng và chuyển) máy chủ lên EC2 Amazon. Nó thay thế hoàn toàn AWS Server Migration Service (SMS — cũ) và là phiên bản được quản lý đầy đủ của CloudEndure Migration.

## 📚 Mục Lục

1. [MGN Là Gì? Lịch Sử Và Bối Cảnh](#mgn-là-gì)
2. [Kiến Trúc Hoạt Động](#kiến-trúc-hoạt-động)
3. [AWS Replication Agent](#aws-replication-agent)
4. [Staging Area — Vùng Dàn Dựng](#staging-area)
5. [Launch Templates — Mẫu Khởi Chạy](#launch-templates)
6. [Nguồn Được Hỗ Trợ](#nguồn-được-hỗ-trợ)
7. [Yêu Cầu Mạng Và Bảo Mật](#yêu-cầu-mạng-và-bảo-mật)
8. [Phí Dịch Vụ](#phí-dịch-vụ)
9. [Hướng Dẫn Thiết Lập Thực Tế](#hướng-dẫn-thiết-lập)
10. [Hạn Chế Cần Biết](#hạn-chế-cần-biết)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 MGN Là Gì? Lịch Sử Và Bối Cảnh

### Lịch Sử AWS Migration Tools

```
Thế hệ 1 (cũ): AWS Server Migration Service (SMS)
├── Ra mắt: 2016
├── Hỗ trợ: VMware, Hyper-V, Azure VMs
├── Cơ chế: Snapshot-based (chụp ảnh định kỳ, không liên tục)
├── Nhược điểm: Replication window dài, downtime lớn
└── Trạng thái: Deprecated (không còn nhận tính năng mới), dùng MGN thay thế

CloudEndure Migration (trung gian):
├── AWS mua lại CloudEndure năm 2019
├── Block-level continuous replication (sao chép liên tục ở cấp block)
├── Miễn phí cho AWS migration
└── Nền tảng kỹ thuật cho MGN

Thế hệ 2 (hiện tại): AWS MGN — Application Migration Service
├── Ra mắt: 2021
├── Dựa trên CloudEndure technology
├── Tích hợp native với AWS ecosystem (IAM, CloudWatch, Migration Hub...)
└── Khuyến nghị sử dụng cho tất cả server rehost migration
```

### MGN Phù Hợp Với Chiến Lược Nào?

MGN được thiết kế chủ yếu cho chiến lược **Rehost** (R2 trong 7Rs) — còn gọi là **Lift-and-Shift**:

| Chiến Lược | MGN Có Phù Hợp? | Ghi Chú |
| ---------- | --------------- | ------- |
| **Rehost** (Rehosting — Nâng và Chuyển) | ✅ Phù hợp nhất | Use case chính của MGN |
| **Replatform** (Tái Nền Tảng) | ⚠️ Một phần | Có thể migrate lên EC2 rồi convert sang container sau |
| **Refactor** (Tái Cấu Trúc) | ❌ Không phù hợp | Refactor cần viết lại code, không phải replication |
| **Relocate** (Di Chuyển Sang Region Khác) | ✅ Hỗ trợ | Migrate từ cloud này sang AWS region khác |

---

## 🏗️ Kiến Trúc Hoạt Động

### Sơ Đồ Tổng Quan

```
[SOURCE SERVER — Server Nguồn]
    │ On-premises / VMware / Hyper-V / Cloud khác
    │
    │ (1) Block-level replication — liên tục qua internet hoặc Direct Connect
    ▼
[AWS STAGING AREA — Vùng Dàn Dựng]
    │ Replication Server (EC2 t3.small — tự động quản lý bởi MGN)
    │ EBS Staging Volumes (lưu data replicated)
    │ → Chi phí thấp: staging server + EBS volumes
    │
    │ (2) Khi sẵn sàng test hoặc cutover
    ▼
[CONVERSION PROCESS — Quá Trình Chuyển Đổi]
    │ Boot mode conversion (BIOS → UEFI nếu cần)
    │ Network driver injection
    │ AWS-specific optimization
    │
    │ (3) Launch theo Launch Template
    ▼
[TARGET EC2 INSTANCE — Instance EC2 Đích]
    │ Region / VPC / Subnet / Security Group theo cấu hình
    │ Instance type theo Launch Template
    └── Test instance (thử nghiệm) hoặc Production instance
```

### Hai Giai Đoạn Replication

**Giai đoạn 1 — Initial Sync (Đồng Bộ Khởi Tạo):**

```
Khi Replication Agent vừa được cài:
├── Quét toàn bộ disk của server nguồn
├── Copy block-by-block lên EBS Staging Volumes
├── Thời gian: phụ thuộc vào dung lượng data và băng thông
│   ├── 100 GB qua 100 Mbps → ~2-3 giờ
│   ├── 1 TB qua 100 Mbps   → ~22-24 giờ
│   └── 5 TB qua 1 Gbps     → ~12-14 giờ
└── Sau khi hoàn thành: chuyển sang Continuous Replication
```

**Giai đoạn 2 — Continuous Replication (Sao Chép Liên Tục):**

```
Sau initial sync:
├── Chỉ sao chép các thay đổi (delta/changed blocks)
├── Độ trễ: thường < 1 giây (near-real-time)
├── Tài nguyên tiêu thụ tối thiểu (low impact trên server nguồn)
└── Tiếp tục cho đến khi:
    ├── Test cutover hoặc Production cutover
    └── Bạn chủ động dừng replication
```

---

## ⚙️ AWS Replication Agent

### Agent Là Gì?

**AWS Replication Agent** — Tác Nhân Sao Chép — là phần mềm nhỏ cài trực tiếp trên server nguồn, chịu trách nhiệm:

1. Capture (bắt) các thay đổi disk ở cấp block (block-level)
2. Nén và mã hóa dữ liệu
3. Truyền an toàn lên AWS Staging Area

### Cài Đặt Agent

```bash
# Trên Linux (RHEL/CentOS/Ubuntu):
wget -O ./aws-replication-installer-init.py \
  https://aws-application-migration-service-<region>.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py

sudo python3 aws-replication-installer-init.py \
  --region ap-southeast-1 \
  --aws-access-key-id AKIAIOSFODNN7EXAMPLE \
  --aws-secret-access-key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  --no-prompt

# Trên Windows (PowerShell — chạy với quyền Administrator):
# Tải installer từ AWS Console → MGN → Add Source Servers
# Chạy .\AwsReplicationWindowsInstaller.exe
```

> **Thực tế:** Trong dự án lớn (100+ server), cài agent thủ công từng máy rất tốn thời gian. Dùng AWS Systems Manager Run Command hoặc Ansible để tự động hóa việc này.

### Yêu Cầu Để Cài Agent

| Yêu Cầu | Chi Tiết |
| ------- | -------- |
| **Quyền trên server nguồn** | Administrator (Windows) hoặc root/sudo (Linux) |
| **Python** | Python 3.6+ (Linux) — thường đã có sẵn |
| **Kết nối mạng** | HTTPS outbound đến AWS endpoint (port 443, 1500) |
| **IAM credentials** | Access Key/Secret hoặc IAM Instance Role với quyền MGN |
| **Disk space** | < 100 MB cho agent binary |

### Vòng Đời Agent (Agent Lifecycle)

```
Trạng thái agent sau khi cài:
Not Ready → (Đang initial sync)
     ↓
Ready for Testing → (Initial sync hoàn thành, sẵn sàng test cutover)
     ↓
Test in Progress → (Đang chạy test cutover)
     ↓
Ready for Cutover → (Test thành công, sẵn sàng production cutover)
     ↓
Cutover in Progress → (Đang thực hiện production cutover)
     ↓
Cutover Complete → (Hoàn thành — server đã migrate xong)
```

---

## 🗄️ Staging Area — Vùng Dàn Dựng

### Staging Area Là Gì?

**Staging Area** — Vùng Dàn Dựng — là môi trường tạm thời trong AWS account của bạn, nơi dữ liệu được replicated trước khi launch thành EC2 production.

```
Staging Area bao gồm:
├── Replication Server (EC2 t3.small)
│   ├── Nhận dữ liệu từ Replication Agent
│   ├── Ghi lên EBS Staging Volumes
│   └── AWS MGN tự động quản lý — bạn không cần cấu hình thủ công
│
└── EBS Staging Volumes
    ├── Số lượng: tương ứng với số disk trên server nguồn
    ├── Loại: gp3 (General Purpose SSD — Ổ Cứng SSD Thông Dụng) theo mặc định
    └── Dung lượng: bằng với disk gốc
```

### Chi Phí Staging Area

```
Ví dụ: Replicate 1 server trong 30 ngày:
├── Replication Server (t3.small):
│   └── ~$0.0208/giờ × 720 giờ = $14.98/tháng
├── EBS Staging Volumes (500 GB gp3):
│   └── $0.08/GB/tháng × 500 GB = $40/tháng
└── MGN Replication Service:
    └── $0.042/server/giờ × 720 giờ = $30.24/tháng
                                       ─────────────────
                           Tổng ≈ $85/tháng/server
```

> **Mẹo tiết kiệm chi phí:** Giảm thiểu thời gian replication bằng cách cài agent, chạy initial sync, test cutover và cutover trong khoảng thời gian ngắn nhất có thể. **Cleanup staging area ngay sau khi cutover hoàn thành.**

---

## 📋 Launch Templates — Mẫu Khởi Chạy

### Launch Template Là Gì?

**Launch Template** — Mẫu Khởi Chạy — là tập hợp cấu hình định nghĩa EC2 instance sẽ được tạo ra sau khi cutover. Đây là nơi bạn "thiết kế" server đích trên AWS.

### Cấu Hình Cần Thiết Trong Launch Template

```
Launch Template cho MGN (cấu hình cho từng server nguồn):

1. Instance Type — Loại Instance
   ├── Ví dụ: m5.xlarge (4 vCPU, 16 GB RAM)
   ├── Dựa vào right-sizing từ ADS/Migration Evaluator
   └── Có thể chọn khác với server gốc (đây là cơ hội optimization)

2. Subnet — Mạng Con
   ├── Chọn VPC và Subnet đích
   └── Private subnet (thường) hoặc Public subnet (nếu cần)

3. Security Groups — Nhóm Bảo Mật
   ├── Mở đúng port cần thiết cho ứng dụng
   └── Thay thế firewall rules của server gốc

4. IAM Instance Profile — Hồ Sơ IAM
   └── Quyền cần thiết cho ứng dụng (truy cập S3, SQS, v.v.)

5. EBS Volume Settings — Cài Đặt Ổ Đĩa
   ├── Volume type: gp3 (khuyến nghị), io1 (IOPS cao)
   ├── Encryption: bật mã hóa KMS nếu cần
   └── Delete on termination: thường để false cho production

6. Tags — Thẻ
   └── Name, Environment, CostCenter, Application...

7. Boot Settings — Cài Đặt Khởi Động
   └── Có thể chuyển từ BIOS → UEFI khi migrate lên instance mới hơn
```

### Launch Template Vs. Launch Settings

```
MGN có hai cấp độ cấu hình:

Account-level Launch Settings (Cài đặt cấp tài khoản):
├── Áp dụng cho TẤT CẢ server trong AWS account
├── Ví dụ: staging subnet, replication server type
└── Là giá trị mặc định nếu không override

Server-level Launch Template (Mẫu cấp server):
├── Cấu hình riêng cho từng server nguồn
├── Override account-level settings
└── Linh hoạt: mỗi server có thể có instance type khác nhau
```

---

## 💻 Nguồn Được Hỗ Trợ

### Hệ Điều Hành Nguồn (Source OS)

```
Linux (64-bit):
├── RHEL — Red Hat Enterprise Linux 6.x, 7.x, 8.x, 9.x
├── CentOS 6.x, 7.x, 8.x
├── Ubuntu 12.04, 14.04, 16.04, 18.04, 20.04, 22.04
├── SUSE Linux Enterprise Server 11 SP4, 12, 15
├── Debian 8, 9, 10, 11
├── Oracle Linux 6.x, 7.x, 8.x
├── Fedora 17-38
└── Amazon Linux, Amazon Linux 2

Windows (64-bit):
├── Windows Server 2008 R2, 2012, 2012 R2, 2016, 2019, 2022
└── Windows 7, 8.1, 10 (enterprise use cases)
```

### Nền Tảng Nguồn (Source Platform)

| Nền Tảng | Hỗ Trợ | Ghi Chú |
| -------- | ------- | ------- |
| **VMware vSphere** | ✅ | Phổ biến nhất — Agent-based |
| **Microsoft Hyper-V** | ✅ | Windows và Linux VMs |
| **Amazon EC2** (region khác) | ✅ | Cross-region migration |
| **Google Cloud Platform** | ✅ | Migrate từ GCP sang AWS |
| **Microsoft Azure** | ✅ | Migrate từ Azure sang AWS |
| **Physical servers** | ✅ | Bare-metal servers |
| **IBM Cloud** | ✅ | Linux workloads |
| **Oracle Cloud** | ✅ | Linux workloads |

> **Lưu ý:** Tất cả đều dùng **agent-based** replication. Không có agentless option cho MGN (khác với ADS có agentless connector).

---

## 🔒 Yêu Cầu Mạng Và Bảo Mật

### Kết Nối Outbound Cần Thiết

```
Server nguồn cần outbound HTTPS đến:

Port 443 (HTTPS):
├── mgn.<region>.amazonaws.com — MGN API
├── s3.<region>.amazonaws.com — S3 (metadata)
└── ec2.<region>.amazonaws.com — EC2 API

Port 1500 (TCP):
└── Replication Server IP — Truyền data replication
    (Cổng 1500 là custom, cần mở outbound trên firewall)
```

> **Quan trọng:** Port 1500 thường bị chặn bởi firewall doanh nghiệp. Đây là nguyên nhân số 1 gây lỗi khi cài agent. Cần phối hợp với team network để mở port này **trước khi cài agent**.

### Mã Hóa Dữ Liệu Khi Truyền

```
Bảo mật trong quá trình replication:
├── TLS 1.2+ mã hóa toàn bộ data in-transit
├── AES-256 mã hóa data at-rest trong EBS Staging Volumes
├── Mỗi server có encryption key riêng
└── Tùy chọn: dùng AWS KMS key của bạn thay vì AWS-managed key
```

### Qua Direct Connect hay Internet?

```
Lựa chọn kết nối:

Qua Internet (mặc định):
├── Đơn giản, không cần cấu hình thêm
├── Dữ liệu được mã hóa TLS
└── Phù hợp cho hầu hết workloads

Qua AWS Direct Connect (khuyến nghị cho workload lớn/nhạy cảm):
├── Băng thông ổn định, không bị ảnh hưởng bởi internet congestion
├── Chi phí transfer thấp hơn internet trong nhiều trường hợp
├── Bảo mật cao hơn (không qua internet public)
└── Cần cấu hình VPC routing để Replication Server nhận traffic từ Direct Connect
```

---

## 💰 Phí Dịch Vụ

### Cơ Cấu Phí MGN

| Thành Phần | Phí | Ghi Chú |
| ---------- | --- | ------- |
| **MGN Replication** | $0.042/server/giờ | Tính từ khi cài agent đến khi cleanup |
| **Replication Server** (EC2) | Theo EC2 pricing (t3.small ~$0.02/giờ) | AWS tự tạo và quản lý |
| **EBS Staging Volumes** | Theo EBS pricing (~$0.08/GB/tháng gp3) | Dung lượng bằng server gốc |
| **Data Transfer** | $0.09/GB ra internet (hoặc theo Direct Connect) | Chỉ tính khi transfer ra ngoài |
| **Test/Production EC2** | Theo EC2 pricing | Chỉ tính khi đang chạy |

### Ví Dụ Tính Phí Thực Tế

```
Kịch bản: Migrate 10 server, mỗi server 500 GB, thực hiện trong 2 tuần

MGN Service:
└── $0.042 × 10 server × (14 ngày × 24 giờ) = $141.12

Replication Servers (10 t3.small):
└── $0.0208 × 10 × 336 giờ = $69.89

EBS Staging Volumes (10 × 500 GB gp3):
└── $0.08 × 5,000 GB × (14/30 tháng) = $186.67

Data transfer (5 TB total, qua internet):
└── $0.09/GB × 5,000 GB = $450

Tổng ước tính: ~$847 cho 10 server trong 2 tuần
→ Khoảng $84/server — so với chi phí downtime và rủi ro migration thủ công: rất xứng đáng
```

---

## 🚀 Hướng Dẫn Thiết Lập Thực Tế

### Bước 1: Khởi Tạo MGN Trong AWS Console

```
AWS Console → MGN → Getting started → Initialize service
├── Chọn Home Region (phải khớp với Migration Hub Home Region)
├── AWS tự tạo IAM roles cần thiết
└── Cấu hình default settings:
    ├── Replication Server instance type: t3.small (mặc định)
    ├── Staging Area Subnet: chọn private subnet trong VPC
    └── Use Private IP: yes (nếu dùng Direct Connect/VPN)
```

### Bước 2: Cài Agent Trên Server Nguồn

```bash
# Bước 2a: Tạo IAM User/Role với quyền MGN:AWSApplicationMigrationAgentPolicy
# Bước 2b: Lấy Access Key ID và Secret Access Key

# Bước 2c: Cài agent (Linux):
sudo python3 aws-replication-installer-init.py \
  --region ap-southeast-1 \
  --aws-access-key-id YOUR_ACCESS_KEY \
  --aws-secret-access-key YOUR_SECRET_KEY \
  --no-prompt

# Sau khi cài thành công:
# Agent tự động đăng ký với MGN Console
# Server xuất hiện trong MGN → Source Servers
# Trạng thái: "Not ready" → "Initial sync" → "Ready for testing"
```

### Bước 3: Cấu Hình Launch Template

```
Trong MGN Console → Source Servers → Chọn server → Launch settings:
├── Chỉnh sửa Launch template
├── Cấu hình:
│   ├── Instance type: dựa trên right-sizing từ Migration Evaluator
│   ├── Subnet: private subnet của application tier
│   ├── Security Group: tạo SG mới hoặc dùng SG có sẵn
│   └── Tags: Name, Environment=production, Application=...
└── Lưu → sẵn sàng cho test cutover
```

---

## ⚠️ Hạn Chế Cần Biết

| Hạn Chế | Chi Tiết |
| ------- | -------- |
| **Agent-only, không agentless** | Phải cài agent trên từng server nguồn — không có connector kiểu ADS |
| **Không hỗ trợ 32-bit OS** | Chỉ hỗ trợ hệ điều hành 64-bit |
| **Không hỗ trợ cluster shared disks** | Windows Server Failover Cluster với shared VHDX không được hỗ trợ |
| **Giới hạn số server** | Soft limit 3 server "in-progress" cùng lúc mỗi Region (có thể tăng) |
| **Dữ liệu EBS Staging không backup** | Staging data không có snapshot backup — nếu staging volume bị xóa, phải sync lại từ đầu |
| **Không tương thích với legacy RAID software** | Một số cấu hình RAID phần mềm đặc biệt không được hỗ trợ |
| **Port 1500 bắt buộc** | Firewall doanh nghiệp thường block cổng này — cần cấu hình trước |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: AWS MGN khác gì với AWS SMS (Server Migration Service cũ)?**

> SMS dùng snapshot-based replication — chụp ảnh định kỳ (mỗi 12-24 giờ), dẫn đến downtime lớn và RPO cao. MGN dùng continuous block-level replication — sao chép liên tục ở cấp block, gần real-time, cho phép downtime chỉ vài phút khi cutover. Ngoài ra, MGN tích hợp native với AWS ecosystem tốt hơn, hỗ trợ nhiều nguồn hơn và có Launch Templates để chuẩn hóa cấu hình. AWS đã deprecated SMS và khuyến nghị tất cả migration dùng MGN.

**Q: Replication Agent trong MGN hoạt động như thế nào?**

> Agent được cài trên server nguồn, capture các thay đổi disk ở cấp block (không phải file-level) theo thời gian thực. Dữ liệu được nén, mã hóa TLS và truyền qua port 1500 đến Replication Server trong AWS Staging Area. Sau initial sync đầy đủ, agent chỉ truyền delta changes — tiêu thụ băng thông và tài nguyên server rất thấp trong giai đoạn ongoing replication.

**Q: Launch Template trong MGN dùng để làm gì?**

> Launch Template định nghĩa cấu hình của EC2 instance sẽ được tạo ra khi test cutover hoặc production cutover: instance type, subnet, security groups, IAM profile, EBS volume settings và tags. Đây là nơi bạn áp dụng right-sizing (thay vì copy y hệt server gốc), thiết lập mạng đúng cho môi trường AWS, và đảm bảo instance được gắn tag theo tiêu chuẩn tổ chức.

**Q: Tại sao MGN yêu cầu port 1500? Có thể đổi không?**

> Port 1500 (TCP) được dùng để truyền dữ liệu replication từ server nguồn đến Replication Server trong Staging Area. Cổng này được chọn để phân biệt với traffic HTTPS thông thường và tránh xung đột với ứng dụng khác. **Không thể thay đổi port này** — đây là yêu cầu kỹ thuật cố định của MGN. Nếu firewall block port 1500, phải làm việc với team network để mở outbound rule cho IP range của AWS Replication Servers trong region tương ứng.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
