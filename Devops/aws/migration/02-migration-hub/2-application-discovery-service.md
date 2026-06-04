# AWS Application Discovery Service — ADS

> **AWS Application Discovery Service (ADS)** tự động thu thập dữ liệu về hạ tầng on-premises: server inventory, process đang chạy, hiệu năng và kết nối mạng — giúp bạn **biết chính xác mình đang có gì** trước khi lên kế hoạch di chuyển.

## 📚 Mục Lục

1. [ADS Là Gì và Tại Sao Cần?](#ads-là-gì-và-tại-sao-cần)
2. [Hai Phương Thức Thu Thập Dữ Liệu](#hai-phương-thức-thu-thập-dữ-liệu)
3. [Agentless Discovery — Khám Phá Không Cần Agent](#agentless-discovery)
4. [Agent-based Discovery — Khám Phá Dùng Agent](#agent-based-discovery)
5. [So Sánh Agentless vs Agent-based](#so-sánh-agentless-vs-agent-based)
6. [Dependency Mapping — Bản Đồ Phụ Thuộc](#dependency-mapping)
7. [Dữ Liệu Thu Thập Được](#dữ-liệu-thu-thập-được)
8. [Tích Hợp Với Các Dịch Vụ Khác](#tích-hợp-với-các-dịch-vụ-khác)
9. [Hướng Dẫn Triển Khai Thực Tế](#hướng-dẫn-triển-khai-thực-tế)
10. [Hạn Chế và Lưu Ý](#hạn-chế-và-lưu-ý)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 ADS Là Gì và Tại Sao Cần?

### Bài Toán Thực Tế

```
Tình huống điển hình tại doanh nghiệp:
├── 500 máy chủ on-premises được quản lý theo kiểu truyền thống
├── Tài liệu (documentation) lỗi thời hoặc không đầy đủ
├── Không ai biết chính xác server A đang kết nối đến server nào
└── Migration plan dựa trên thông tin không chắc chắn → rủi ro cao
```

**ADS giải quyết bằng cách:**

1. **Tự động** thu thập dữ liệu — không cần hỏi từng admin
2. **Khách quan** — dữ liệu thực tế từ hệ thống, không dựa trên tài liệu cũ
3. **Toàn diện** — bắt được cả các kết nối ẩn giữa services
4. **Lưu trữ dài hạn** — thu thập nhiều tuần để hiểu usage patterns thực sự

### Dữ Liệu ADS Dùng Để Làm Gì?

```
Dữ liệu ADS →
    ├── Migration Hub: Inventory và theo dõi tiến trình
    ├── Migration Evaluator: Phân tích TCO và right-sizing
    ├── AWS Athena: Query dữ liệu discovery để báo cáo tùy chỉnh
    └── Bản đồ phụ thuộc: Lên kế hoạch Migration Waves
```

---

## 🔄 Hai Phương Thức Thu Thập Dữ Liệu

ADS cung cấp hai phương thức hoàn toàn khác nhau về cách cài đặt và độ sâu dữ liệu:

```
                  ADS
                   │
       ┌───────────┴───────────┐
       │                       │
  Agentless                Agent-based
  Discovery                Discovery
  Connector                Agent
       │                       │
  Cài trên VMware          Cài trên từng
  vCenter dưới dạng        máy chủ (Windows/Linux)
  virtual appliance
       │                       │
  Thu thập metadata         Thu thập đầy đủ:
  VM từ hypervisor          process, network
  (không cần vào VM)        connections, performance
```

---

## 🖥️ Agentless Discovery — Khám Phá Không Cần Agent {#agentless-discovery}

### Cơ Chế Hoạt Động

**Agentless Discovery Connector** — Trình Thu Thập Không Cần Agent — là một **OVA (Open Virtual Appliance)** — file máy ảo — được cài đặt trên VMware vCenter.

```
Cách hoạt động:
VMware vCenter
    ↓ API calls (VMware API)
Agentless Connector (OVA trên ESXi)
    ↓ HTTPS (port 443)
AWS Application Discovery Service
    ↓
Migration Hub / Migration Evaluator
```

**Connector giao tiếp với vCenter bằng VMware API**, đọc thông tin từ hypervisor mà không cần đăng nhập vào từng VM.

### Yêu Cầu Cài Đặt

| Yêu Cầu | Chi Tiết |
| ------- | -------- |
| **Môi trường** | VMware vCenter Server 5.5, 6.0, 6.5, 6.7, 7.0 |
| **Phần cứng tối thiểu** | 4 vCPU, 8 GB RAM, 20 GB disk |
| **Kết nối** | Connector phải ra được internet qua HTTPS port 443 |
| **Quyền truy cập** | Tài khoản read-only trên vCenter là đủ |

### Dữ Liệu Thu Thập Được (Agentless)

```
Metadata từ VMware:
├── VM Name, Guest OS, VM tools version
├── Số CPU, RAM (allocated)
├── Disk size và VMDK location
├── IP address, MAC address
├── Power state (on/off)
├── vCenter cluster và host placement
└── Performance metrics: CPU utilization, memory utilization, disk I/O (từ vCenter)
```

> **Giới hạn quan trọng:** Agentless connector **không** thấy được process đang chạy bên trong VM, và **không** thu thập network connections giữa VMs. Đây là điểm yếu lớn nhất.

### Ưu Điểm

- **Nhanh triển khai:** Chỉ cài một OVA duy nhất, không cần vào từng server
- **Ít can thiệp:** Không cần thay đổi gì trên production servers
- **Phù hợp môi trường VMware lớn:** Có thể thu thập hàng nghìn VMs nhanh chóng

---

## 🤖 Agent-based Discovery — Khám Phá Dùng Agent {#agent-based-discovery}

### Cơ Chế Hoạt Động

**AWS Discovery Agent** là phần mềm cài trực tiếp **trên từng server** (physical hoặc virtual). Agent chạy nền và liên tục thu thập dữ liệu về hệ thống.

```
Cách hoạt động:
Từng Server (Windows/Linux)
    │
    ├── AWS Discovery Agent (chạy nền)
    │       ↓ thu thập mỗi 15 phút
    │   ├── Process list (danh sách tiến trình)
    │   ├── Network connections (kết nối TCP/UDP)
    │   ├── CPU/RAM/Disk metrics chi tiết
    │   └── Installed software
    │       ↓ HTTPS port 443
    └── AWS Application Discovery Service
            ↓
        Migration Hub / Athena
```

### Hệ Điều Hành Được Hỗ Trợ

| Hệ Điều Hành | Phiên Bản Hỗ Trợ |
| ------------ | ----------------- |
| **Windows Server** | 2008 R2, 2012, 2012 R2, 2016, 2019, 2022 |
| **Amazon Linux** | 2012.03 trở lên |
| **Ubuntu** | 12.04, 14.04, 16.04, 18.04, 20.04 |
| **Red Hat Enterprise Linux** | 6.x, 7.x, 8.x |
| **CentOS** | 6.x, 7.x, 8.x |
| **SUSE Linux Enterprise Server** | 11 SP3, 12 |

### Dữ Liệu Thu Thập Được (Agent-based)

```
Dữ liệu hệ thống:
├── Hostname, FQDN (Fully Qualified Domain Name)
├── OS name, version, kernel version
├── CPU: số core, model, utilization (mỗi 15 phút)
├── RAM: tổng, đang dùng, utilization
├── Disk: partitions, capacity, utilization, IOPS
└── Network interfaces: IP, MAC, bandwidth

Dữ liệu process (quan trọng nhất):
├── Danh sách tất cả process đang chạy
├── Process name, PID (Process ID), command line
└── Tài nguyên từng process đang tiêu thụ

Dữ liệu kết nối mạng (quan trọng nhất):
├── TCP connections: source IP:port → destination IP:port
├── UDP connections (best-effort)
├── Inbound và outbound connections
└── Tần suất kết nối (dùng để xác định dependency quan trọng)

Phần mềm đã cài:
├── Danh sách installed packages (Windows: Add/Remove Programs; Linux: rpm/dpkg)
└── Version numbers
```

### Ưu Điểm

- **Dependency mapping chính xác:** Biết chính xác server nào kết nối đến server nào
- **Process-level visibility:** Biết ứng dụng gì đang chạy
- **Không phụ thuộc VMware:** Hoạt động trên physical servers, Hyper-V, bare-metal

---

## ⚖️ So Sánh Agentless vs Agent-based

| Tiêu Chí | Agentless Connector | Discovery Agent |
| -------- | -------------------- | --------------- |
| **Cài đặt** | 1 OVA trên vCenter | Cài trên từng server |
| **Yêu cầu môi trường** | Chỉ VMware vCenter | Windows / Linux bất kỳ |
| **Can thiệp vào server** | Không | Phải cài agent |
| **Server metadata** | ✅ Có | ✅ Có |
| **CPU/RAM metrics** | ✅ Có (từ vCenter) | ✅ Có (chi tiết hơn) |
| **Process list** | ❌ Không | ✅ Có |
| **Network connections** | ❌ Không | ✅ Có |
| **Dependency mapping** | ❌ Không chính xác | ✅ Chính xác |
| **Physical servers** | ❌ Không hỗ trợ | ✅ Có |
| **Thời gian triển khai** | Nhanh (1-2 giờ) | Chậm hơn (cài từng server) |
| **Phù hợp cho** | VMware lớn, muốn inventory nhanh | Cần dependency map chi tiết |

### Khi Nào Dùng Loại Nào?

```
Chọn Agentless Connector nếu:
├── Toàn bộ môi trường là VMware vCenter
├── Cần inventory nhanh (1-2 ngày setup)
├── Chỉ cần metadata cơ bản (hostname, CPU, RAM, disk)
└── Không thể cài phần mềm lên production servers

Chọn Discovery Agent nếu:
├── Có physical servers hoặc non-VMware hypervisor
├── Cần dependency map để lên kế hoạch Migration Waves
├── Cần biết process nào đang chạy (license audit, consolidation)
└── Đang dùng chung với Migration Evaluator (cần performance data chi tiết)

Kết hợp cả hai (phổ biến nhất):
├── Dùng Agentless trên tất cả VMware VMs → inventory nhanh
└── Dùng Agent trên các "tier servers" quan trọng → dependency mapping
```

---

## 🗺️ Dependency Mapping — Bản Đồ Phụ Thuộc

### Tại Sao Dependency Mapping Quan Trọng?

```
Ví dụ: Application "E-Commerce Platform"
Nếu không có dependency map:
└── Bạn di chuyển "web-01" lên AWS → ứng dụng lỗi
    Nguyên nhân: web-01 kết nối đến cache-redis-02 (bạn không biết)
    cache-redis-02 vẫn đang on-premises
    Kết quả: rollback, mất 1 ngày khắc phục

Nếu có dependency map từ ADS Agent:
└── Bạn thấy: web-01 → cache-redis-02 → db-mysql-01 → db-replica-02
    Kế hoạch migration đúng đắn:
    Wave 1: db-mysql-01 + db-replica-02 (migrate DB trước)
    Wave 2: cache-redis-02 (cache sau DB)
    Wave 3: web-01 (web tier cuối cùng)
```

### Cách ADS Tạo Dependency Map

ADS Discovery Agent thu thập **tất cả TCP connections** giữa các server. Khi phân tích:

```
Dữ liệu raw từ Agent:
├── web-01:52341 → db-mysql-01:3306 (TCP, 847 connections/ngày)
├── web-01:43210 → cache-redis-02:6379 (TCP, 12,400 connections/ngày)
├── db-mysql-01:54123 → db-replica-02:3306 (TCP, continuous replication)
└── monitoring-01:161 → web-01:161 (SNMP monitoring)

Sau khi phân tích → Dependency Map:
web-01
├── [Critical] → db-mysql-01:3306 (MySQL)
├── [Critical] → cache-redis-02:6379 (Redis)
└── [Monitoring] → monitoring-01 (ít quan trọng)

db-mysql-01
└── [Replication] → db-replica-02:3306
```

### Visualize Dependency Map

Dữ liệu ADS có thể export sang:
- **Migration Hub** — network visualization tích hợp sẵn
- **AWS Athena** — query SQL tùy chỉnh để phân tích
- **Third-party tools** — Lucidchart, draw.io (import CSV từ ADS)

---

## 📦 Dữ Liệu Thu Thập Được — Tóm Tắt Đầy Đủ

### Dữ Liệu Gửi Về ADS

Tất cả dữ liệu được mã hóa (AES-256) khi truyền và lưu trữ:

```
Dữ liệu lưu trong ADS (dùng Athena để query):

s3://aws-application-discovery-service-{account}/
├── servers/            → Metadata từng server
├── processes/          → Process list (agent-based only)
├── connections/        → Network connections (agent-based only)
└── system-performance/ → CPU/RAM/Disk metrics
```

### Retention (Thời Gian Lưu Trữ)

- ADS lưu dữ liệu trong **90 ngày** kể từ lần thu thập cuối
- Sau khi xóa agent/connector, dữ liệu vẫn được giữ 90 ngày
- Có thể export ra S3 để lưu lâu hơn

---

## 🔗 Tích Hợp Với Các Dịch Vụ Khác

```
ADS
├── → Migration Hub
│       Tự động đồng bộ server inventory
│       Server trong ADS hiển thị trong Migration Hub Dashboard
│
├── → Migration Evaluator
│       Import dữ liệu ADS (hoặc dữ liệu upload thủ công)
│       Phân tích performance data để đề xuất right-sizing
│
├── → Amazon Athena
│       Query dữ liệu raw từ ADS bằng SQL
│       Ví dụ: "Show me all servers with CPU > 80% average"
│       Ví dụ: "Find all connections to port 1433 (SQL Server)"
│
└── → AWS Lake Formation (tuỳ chọn)
        Tổng hợp dữ liệu discovery vào Data Lake
        Kết hợp với dữ liệu business khác để phân tích
```

### Export Dữ Liệu Để Dùng Bên Ngoài

ADS hỗ trợ export dữ liệu dưới dạng CSV:

```
Ví dụ dữ liệu export (servers.csv):
ServerID, Hostname, IPAddress, OS, CPUCores, RAMgb, DiskGB, AvgCPU%, AvgRAM%
srv-001, web-app-01, 10.0.1.10, Windows Server 2019, 8, 32, 200, 35.2, 68.4
srv-002, db-mysql-01, 10.0.1.20, Amazon Linux 2, 16, 64, 1000, 42.1, 78.9
```

---

## 🚀 Hướng Dẫn Triển Khai Thực Tế

### Triển Khai Agentless Connector (VMware)

```
Bước 1: Download OVA
→ AWS Console → Application Discovery Service → Data collection → Connectors
→ Download connector OVA (~500 MB)

Bước 2: Deploy OVA trên ESXi/vCenter
→ vCenter → Deploy OVF Template → chọn file OVA
→ Cấu hình: 4 vCPU, 8 GB RAM, 20 GB disk
→ Gắn vào network có thể ra internet (port 443)

Bước 3: Cấu hình Connector
→ Truy cập IP của Connector qua web browser
→ Đăng nhập: admin/admin (đổi ngay)
→ Nhập: vCenter hostname, username, password (read-only account)
→ Nhập: AWS credentials (IAM user với AWSApplicationDiscoveryServiceFullAccess)
→ Chọn: Home Region của Migration Hub

Bước 4: Bắt đầu thu thập
→ ADS Console → Start data collection
→ Dữ liệu xuất hiện sau 15-30 phút
```

### Triển Khai Discovery Agent (Windows/Linux)

```
Cài đặt trên Windows (PowerShell):
# Download installer
Invoke-WebRequest -Uri https://s3.us-west-2.amazonaws.com/aws-discovery-agent.us-west-2/windows/latest/AWSDiscoveryAgentInstaller.msi -OutFile AWSDiscoveryAgentInstaller.msi

# Cài đặt với AWS credentials
msiexec.exe /i AWSDiscoveryAgentInstaller.msi REGION="ap-southeast-1" KEY_ID="AKIA..." KEY_SECRET="..." /q

Cài đặt trên Linux (bash):
# Download
curl -o /tmp/aws-discovery-agent.tar.gz https://s3.us-west-2.amazonaws.com/aws-discovery-agent.us-west-2/linux/latest/aws-discovery-agent.tar.gz

# Cài đặt
tar -xzf /tmp/aws-discovery-agent.tar.gz
sudo bash install -r ap-southeast-1 -k AKIA... -s SECRET_KEY

# Kiểm tra trạng thái
sudo systemctl status aws-discovery-daemon
```

### Triển Khai Hàng Loạt (Mass Deployment)

Với hàng trăm server, không thể cài tay từng cái:

```
Cách tự động hóa:
├── Windows: AWS Systems Manager Run Command
│       → ssm:SendCommand → AWSDiscoveryAgentInstall
├── Linux: Ansible playbook (cài agent qua yum/apt)
├── Puppet/Chef: Module tự động cài agent
└── AWS OpsWorks: Layer-based deployment
```

---

## ⚠️ Hạn Chế và Lưu Ý

| Hạn Chế | Chi Tiết |
| ------- | -------- |
| **Agentless chỉ cho VMware** | Không hỗ trợ Hyper-V, Xen, KVM natively |
| **Agent cần internet** | Port 443 phải mở từ server ra ngoài (hoặc qua VPC endpoint) |
| **Không real-time** | Dữ liệu có độ trễ 15-30 phút |
| **90 ngày retention** | Sau 90 ngày không active, dữ liệu bị xóa |
| **Không thu thập nội dung** | ADS **không** đọc nội dung file, database, hay application data — chỉ metadata hệ thống |
| **Agent overhead** | Agent tiêu thụ ~1% CPU, ~10 MB RAM — thường không đáng kể |

### Bảo Mật Dữ Liệu

```
Dữ liệu nhạy cảm:
├── ADS KHÔNG thu thập: mật khẩu, key, nội dung database, file
├── ADS thu thập: hostname, IP, process names, port numbers
│
Mã hóa:
├── In transit: TLS 1.2 (HTTPS)
├── At rest: AES-256 trong ADS storage
└── Tất cả dữ liệu thuộc về AWS Account của bạn
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Giải thích sự khác biệt giữa Agentless Connector và Discovery Agent trong ADS?**

> Agentless Connector là một virtual appliance (OVA) cài trên VMware vCenter — nó đọc metadata VM từ hypervisor mà không cần đụng vào từng server. Thu thập được: hostname, CPU, RAM, disk, OS. Nhược điểm: không thấy process bên trong VM, không thấy network connections. Discovery Agent ngược lại — cài trực tiếp trên từng server, thu thập đầy đủ process list, TCP connections, và performance metrics chi tiết. Chọn Agentless khi cần inventory nhanh trên VMware; chọn Agent khi cần dependency map chính xác.

**Q: Làm thế nào ADS giúp lên kế hoạch Migration Waves?**

> Discovery Agent thu thập tất cả TCP connections giữa servers trong nhiều tuần. Từ dữ liệu đó, ADS tạo dependency map: server A kết nối đến B, C; B kết nối đến D. Khi lên kế hoạch Migration Waves, các server có dependency mạnh với nhau phải nằm cùng một wave hoặc wave liền kề (migrate servers phụ thuộc trước). Nếu migrate server A nhưng bỏ B lại, A sẽ mất kết nối và fail.

**Q: ADS có thu thập dữ liệu nhạy cảm không?**

> Không. ADS chỉ thu thập metadata hệ thống: hostname, IP, OS version, process names, port numbers, CPU/RAM/disk metrics. Nó không đọc nội dung database, không đọc file, không thu thập mật khẩu hay credentials. Tất cả dữ liệu được mã hóa khi truyền (TLS 1.2) và khi lưu trữ (AES-256).

**Q: ADS khác gì với AWS Migration Evaluator?**

> ADS là công cụ **thu thập dữ liệu** — nó đóng vai trò như "máy đo lường" hạ tầng on-premises. Migration Evaluator là công cụ **phân tích** — nó nhập dữ liệu (từ ADS hoặc từ RVTools/SCOM), rồi tính toán TCO và đề xuất instance type tối ưu trên AWS. Trong quy trình assessment: ADS chạy trước (2-4 tuần), sau đó export dữ liệu vào Migration Evaluator để ra báo cáo.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
