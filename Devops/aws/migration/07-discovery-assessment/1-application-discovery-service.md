# AWS Application Discovery Service (ADS) — Khám Phá Hạ Tầng On-Premises Chi Tiết

> **AWS Application Discovery Service (ADS)** là dịch vụ giúp bạn lập kế hoạch migration bằng cách tự động thu thập thông tin về hạ tầng on-premises (tại chỗ): máy chủ, ứng dụng đang chạy, hiệu năng, và quan trọng nhất — **dependency map** (bản đồ phụ thuộc) giữa các ứng dụng với nhau. Không có ADS, bạn sẽ phải điền inventory (kiểm kê) thủ công — tốn thời gian và dễ bỏ sót, dẫn đến migration bị gián đoạn.

## 📚 Mục Lục (Table of Contents)

1. [ADS Là Gì? Giải Quyết Vấn Đề Gì?](#ads-là-gì-giải-quyết-vấn-đề-gì)
2. [Hai Phương Thức Thu Thập: Agentless vs Agent-Based](#hai-phương-thức-thu-thập-agentless-vs-agent-based)
3. [Agentless Collector — Chi Tiết](#agentless-collector--chi-tiết)
4. [Discovery Agent — Chi Tiết](#discovery-agent--chi-tiết)
5. [Dữ Liệu Được Thu Thập](#dữ-liệu-được-thu-thập)
6. [Dependency Analysis — Phân Tích Phụ Thuộc](#dependency-analysis--phân-tích-phụ-thuộc)
7. [Migration Hub Integration — Tích Hợp Migration Hub](#migration-hub-integration--tích-hợp-migration-hub)
8. [Quy Trình Triển Khai Thực Tế](#quy-trình-triển-khai-thực-tế)
9. [Export Dữ Liệu Và Phân Tích](#export-dữ-liệu-và-phân-tích)
10. [Bảo Mật Và Quyền Riêng Tư](#bảo-mật-và-quyền-riêng-tư)
11. [Hạn Chế Và Lưu Ý](#hạn-chế-và-lưu-ý)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 ADS Là Gì? Giải Quyết Vấn Đề Gì?

### Bài Toán Thực Tế

```
TÌNH HUỐNG PHỔ BIẾN:

Bạn được yêu cầu lên kế hoạch di chuyển 300 máy chủ lên AWS.

Câu hỏi cần trả lời:
├── Có bao nhiêu máy chủ? OS là gì? Bao nhiêu CPU/RAM?
├── Ứng dụng nào đang chạy trên mỗi máy?
├── App A có kết nối đến DB B không? Nếu có → phải di chuyển cùng nhau
├── Máy nào dùng ít → có thể Retire (loại bỏ) hoặc Rightsize nhỏ lại?
├── Hiệu năng thực tế là bao nhiêu? (không phải capacity tối đa)
└── Chi phí on-premises thực sự là bao nhiêu?

Nếu làm thủ công:
├── Liên hệ từng team để hỏi → mất 2-3 tháng
├── Dữ liệu không nhất quán, outdated (lỗi thời)
├── Bỏ sót các kết nối ngầm (undocumented dependencies)
└── Tốn nhân lực cao, dễ sai

Với AWS ADS:
├── Triển khai Agentless Collector → 1-2 ngày setup
├── Chạy thu thập 2-4 tuần → dữ liệu tự động, chính xác
├── Dependency map được vẽ tự động từ network connections
└── Export vào Migration Evaluator và Migration Hub
```

### Vị Trí Của ADS Trong Migration Journey

```
AWS MIGRATION JOURNEY:

Phase 1: ASSESS (Đánh Giá)
├── ✅ ADS ← CHÚNG TA ĐANG Ở ĐÂY
├── Migration Evaluator
└── Well-Architected Review

Phase 2: MOBILIZE (Chuẩn Bị)
├── Migration Hub setup
├── Landing Zone (môi trường AWS nền tảng)
└── Pilot migrations

Phase 3: MIGRATE & MODERNIZE (Di Chuyển & Hiện Đại Hóa)
├── AWS MGN (Application Migration Service)
├── AWS DMS (Database Migration Service)
└── DataSync / Snow Family
```

---

## ⚖️ Hai Phương Thức Thu Thập: Agentless vs Agent-Based

```
┌──────────────────────────────────┬──────────────────────────────────────────┐
│ AGENTLESS COLLECTOR              │ DISCOVERY AGENT                          │
│ (Thu Thập Không Cần Agent)       │ (Agent Khám Phá)                         │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Cơ chế hoạt động:                │ Cơ chế hoạt động:                        │
│ Máy ảo OVA cài trên vSphere      │ Phần mềm cài trực tiếp trên OS           │
│ → Đọc API VMware vCenter         │ → Gửi telemetry lên ADS qua HTTPS        │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Yêu cầu môi trường:              │ Yêu cầu môi trường:                      │
│ • VMware vSphere 5.x trở lên     │ • Windows Server 2008 R2 trở lên         │
│ • vCenter credentials (xác thực) │ • Linux (các distro phổ biến)            │
│                                  │ • Cần quyền admin/root                   │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Dữ liệu thu thập được:           │ Dữ liệu thu thập được:                   │
│ • VM config: vCPU, RAM, Disk     │ TẤT CẢ của Agentless CỘNG THÊM:         │
│ • VM state: running/stopped      │ • Running processes (tiến trình đang     │
│ • Network adapters (card mạng)   │   chạy): tên, PID, user, CPU/RAM        │
│ • Host/datastore (kho lưu trữ)   │ • Network connections: IP:port pairs     │
│   associations                   │   (kết nối mạng chi tiết theo TCP/UDP)  │
│ • Average CPU/RAM utilization    │ • Disk I/O (tốc độ đọc/ghi)             │
│   (utilization trung bình)       │ • Installed software inventory           │
│                                  │   (danh sách phần mềm đã cài)           │
│                                  │ • System configuration chi tiết          │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Dependency mapping:              │ Dependency mapping:                      │
│ Network-level (IP flows)         │ Process-level + Network-level            │
│ → Độ chính xác: TRUNG BÌNH      │ → Độ chính xác: CAO                     │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Thời gian setup:                 │ Thời gian setup:                         │
│ 1-2 ngày (cài OVA + cấu hình)   │ Vài ngày đến vài tuần (tùy số server)   │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ PHÙ HỢP KHI:                    │ PHÙ HỢP KHI:                            │
│ ✅ VMware environment           │ ✅ Cần dependency map chính xác           │
│ ✅ Muốn inventory nhanh         │ ✅ Bare-metal servers (máy vật lý)        │
│ ✅ Không thể cài agent           │ ✅ Windows không có vSphere              │
│ ✅ Large-scale: 1000+ VMs        │ ✅ Ứng dụng quan trọng, cần phân tích kỹ│
└──────────────────────────────────┴──────────────────────────────────────────┘

BEST PRACTICE (Thực hành tốt nhất):
Dùng CẢ HAI — Agentless để có inventory tổng quát nhanh,
Discovery Agent cho các ứng dụng tier-1 (quan trọng nhất) cần dependency map chính xác.
```

---

## 🖥️ Agentless Collector — Chi Tiết

### Kiến Trúc Agentless Collector

```
KIẾN TRÚC AGENTLESS COLLECTOR:

  On-Premises Environment (Môi Trường On-Premises)
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  VMware vCenter                                      │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐        │
  │  │  VM 001  │    │  VM 002  │    │  VM 003  │        │
  │  │ web-app  │    │ app-srvr │    │  mysql   │        │
  │  └──────────┘    └──────────┘    └──────────┘        │
  │       └──────────────┴──────────────┘                │
  │                      │ vCenter API (VMware API)      │
  │                      ▼                               │
  │  ┌───────────────────────────────────┐               │
  │  │   Agentless Collector (OVA VM)    │               │
  │  │   ─────────────────────────────   │               │
  │  │   • Thu thập qua vCenter API      │               │
  │  │   • Không cần cài gì lên VM       │               │
  │  │   • Cần: vCenter credentials      │               │
  │  └───────────────────┬───────────────┘               │
  └──────────────────────│───────────────────────────────┘
                         │ HTTPS (port 443)
                         ▼
                  AWS ADS Service (Cloud)
                  ──────────────────────
                  Lưu trữ và phân tích data
```

### Quy Trình Cài Đặt Agentless Collector

```
BƯỚC 1: Tải OVA file từ ADS Console
├── Vào AWS Console → Migration & Transfer → Application Discovery Service
├── Chọn "Set up agentless discovery"
└── Download file .ova (khoảng 1-2 GB)

BƯỚC 2: Import OVA vào vSphere
├── Mở vSphere Client → Deploy OVF Template
├── Upload file .ova đã tải
├── Cấu hình network (cần HTTPS ra ngoài, port 443)
└── Khởi động VM

BƯỚC 3: Cấu hình Collector
├── Truy cập web UI của Collector VM (IP nội bộ)
├── Nhập AWS credentials (hoặc IAM role)
├── Nhập vCenter IP/hostname, username, password
└── Bắt đầu thu thập (collection sẽ tự động)

BƯỚC 4: Xác Nhận Dữ Liệu Về ADS
├── Vào ADS Console → Servers
├── Kiểm tra danh sách server đã xuất hiện chưa
└── Chờ 24-48 giờ để có đủ dữ liệu ban đầu

BƯỚC 5: Chạy Thu Thập 2-4 Tuần
└── Đủ dữ liệu để tính average + peak utilization chính xác
```

---

## 🤖 Discovery Agent — Chi Tiết

### Cách Hoạt Động

```
DISCOVERY AGENT HOẠT ĐỘNG NHƯ SAU:

Server On-Premises
┌─────────────────────────────────────────────┐
│  Operating System (Hệ Điều Hành)            │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │     AWS Discovery Agent              │   │
│  │     ─────────────────────────────    │   │
│  │     Thu thập mỗi 15 phút:            │   │
│  │     • ps aux (running processes)     │   │
│  │     • netstat -an (TCP/UDP conns)    │   │
│  │     • top (CPU, RAM real-time)       │   │
│  │     • df -h (disk usage)             │   │
│  │     • dpkg/rpm list (installed pkgs) │   │
│  └──────────────────┬───────────────────┘   │
└─────────────────────│───────────────────────┘
                      │ HTTPS port 443 (encrypted)
                      ▼
              AWS ADS Cloud Service
              ─────────────────────
              Lưu trữ, phân tích,
              vẽ dependency map
```

### Cài Đặt Discovery Agent

```bash
# TRÊN LINUX (Ubuntu/CentOS/RHEL):

# Bước 1: Download agent installer
curl -O https://s3.us-west-2.amazonaws.com/aws-discovery-agent.us-west-2/linux/latest/aws-discovery-agent.tar.gz

# Bước 2: Giải nén
tar -xzf aws-discovery-agent.tar.gz

# Bước 3: Cài đặt
sudo bash install -r <AWS_REGION> -k <ACCESS_KEY_ID> -s <SECRET_ACCESS_KEY>

# Bước 4: Kiểm tra agent đang chạy
sudo systemctl status aws-discovery-daemon

# TRÊN WINDOWS SERVER:
# Tải .msi installer từ ADS Console
# Chạy với quyền Administrator
# Nhập AWS credentials khi được yêu cầu
```

### Tần Suất Thu Thập

```
AGENT THU THẬP DỮ LIỆU VỚI TẦN SUẤT:

├── System metrics (CPU, RAM, Disk I/O):    Mỗi 15 phút
├── Network connections (TCP/UDP states):  Mỗi 15 phút
├── Running processes:                      Mỗi 15 phút
├── Installed software list:               Mỗi 6 giờ
└── System configuration:                  Mỗi 6 giờ

Dữ liệu được buffer (đệm) cục bộ và gửi lên ADS:
├── Nếu internet ổn định: gửi gần real-time
└── Nếu mất kết nối: buffer lên đến 5 ngày cục bộ, gửi khi có lại
```

---

## 📊 Dữ Liệu Được Thu Thập

### Server Configuration Data (Dữ Liệu Cấu Hình Máy Chủ)

```
THÔNG TIN CẤU HÌNH MÁY CHỦ:

├── Định danh:
│   ├── Hostname (tên máy chủ)
│   ├── IP addresses (IPv4, IPv6)
│   ├── MAC addresses
│   └── Server type: physical/virtual (vật lý/ảo)
│
├── Cấu hình tài nguyên:
│   ├── Number of CPUs (số nhân CPU)
│   ├── CPU speed (tốc độ CPU, MHz)
│   ├── Total RAM (GB)
│   ├── Disk capacity per volume (GB)
│   └── Network adapters count
│
└── Hệ điều hành:
    ├── OS family: Windows / Linux
    ├── OS name: "Windows Server 2019", "Ubuntu 20.04 LTS"
    ├── OS version và patch level
    └── Kernel version (Linux)
```

### Performance Data (Dữ Liệu Hiệu Năng) — Quan Trọng Cho Right-Sizing

```
PERFORMANCE METRICS (Chỉ Số Hiệu Năng):

CPU Utilization (Mức Sử Dụng CPU):
├── Average (trung bình) theo ngày, tuần, tháng
├── Peak (cao điểm) — quan trọng để không undersizing
└── P95 — phần trăm thứ 95, dùng cho right-sizing

RAM Utilization (Mức Sử Dụng RAM):
├── Used RAM (GB) trung bình
└── Peak RAM usage

Disk I/O (Hoạt Động Đọc/Ghi Đĩa):
├── Read throughput (MB/s)
├── Write throughput (MB/s)
└── IOPS (Input/Output Operations Per Second — Số Thao Tác Đọc/Ghi Mỗi Giây)

Network throughput:
├── Inbound (MB/s) — dữ liệu vào
└── Outbound (MB/s) — dữ liệu ra

WHY RIGHT-SIZING MATTERS (Tại Sao Right-Sizing Quan Trọng):
├── Nếu server on-premises: 32 vCPU, 128 GB RAM
├── Nhưng thực tế chỉ dùng: 4 vCPU avg (25%), 32 GB RAM (25%)
├── AWS instance phù hợp: m5.2xlarge (8 vCPU, 32 GB) thay vì x1e.4xlarge
└── Tiết kiệm: ~70% chi phí EC2
```

### Process & Network Data (Agent Only — Chỉ Có Khi Dùng Agent)

```
RUNNING PROCESSES (Tiến Trình Đang Chạy):

Ví dụ output:
├── java -jar /opt/app/api-server-1.2.jar (PID 1234, user: appuser, CPU: 15%, RAM: 2.4 GB)
├── mysqld --defaults-file=/etc/mysql/my.cnf (PID 5678, user: mysql, CPU: 8%, RAM: 4.1 GB)
├── nginx: master process (PID 9012, user: www-data, CPU: 1%, RAM: 0.1 GB)
└── ...

→ ADS biết: server này chạy Java application, MySQL database, và Nginx web server

NETWORK CONNECTIONS (Kết Nối Mạng):

Ví dụ output:
├── TCP 192.168.1.10:8080 → 192.168.1.20:3306 (ESTABLISHED)
│   → Web server kết nối đến MySQL
├── TCP 192.168.1.20:8080 → 192.168.1.40:6379 (ESTABLISHED)
│   → App server kết nối đến Redis
└── TCP 192.168.1.10:443 → 203.0.113.50:443 (ESTABLISHED)
    → Web server kết nối đến external API

→ ADS vẽ được dependency map từ thông tin này
```

---

## 🗺️ Dependency Analysis — Phân Tích Phụ Thuộc

### Dependency Visualization (Trực Quan Hóa Phụ Thuộc)

```
ADS DEPENDENCY MAP — VÍ DỤ THỰC TẾ:

Hệ thống E-Commerce (Thương Mại Điện Tử):

┌─────────────────────────────────────────────────────────────────────────┐
│                          INTERNET                                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Load Balancer      │
                    │   (HAProxy)          │
                    │   192.168.1.5        │
                    └────────┬────────────┘
                             │ TCP:80/443
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──┐  ┌────────▼───┐  ┌──────▼──────┐
    │ Web-01     │  │ Web-02     │  │ Web-03      │
    │ Nginx      │  │ Nginx      │  │ Nginx       │
    │ .1.10      │  │ .1.11      │  │ .1.12       │
    └─────┬──────┘  └──────┬─────┘  └──────┬──────┘
          │                │               │
          └────────────────┼───────────────┘
                           │ TCP:8080
              ┌────────────┼────────────┐
              │            │            │
    ┌─────────▼──┐  ┌──────▼──────┐
    │ App-01     │  │ App-02     │
    │ Java API   │  │ Java API   │
    │ .1.20      │  │ .1.21      │
    └──┬─────────┘  └──┬──────────┘
       │               │
       ├───────────────┤
       │ TCP:3306      │ TCP:6379
       │               │
┌──────▼──────┐  ┌─────▼───────┐  ┌──────────────┐
│ MySQL       │  │ Redis Cache │  │ RabbitMQ     │
│ Primary     │  │ .1.40       │  │ (Message     │
│ .1.30       │  └─────────────┘  │  Queue)      │
└──────┬──────┘                   │ .1.50        │
       │                          └──────────────┘
       │ Replication
┌──────▼──────┐
│ MySQL       │
│ Replica     │
│ .1.31       │
└─────────────┘

KẾT LUẬN TỪ DEPENDENCY MAP:
Migration Wave 1 (Đợt Di Chuyển 1):
  → MySQL Primary + Replica + Redis + RabbitMQ
  → Lý do: App servers phụ thuộc vào các thành phần này

Migration Wave 2 (Đợt Di Chuyển 2):
  → App-01 + App-02
  → Lý do: Web servers phụ thuộc vào App servers

Migration Wave 3 (Đợt Di Chuyển 3):
  → Web-01 + Web-02 + Web-03 + Load Balancer
  → Lý do: Thành phần cuối cùng, phụ thuộc vào tất cả bên dưới
```

### Application Grouping (Nhóm Ứng Dụng)

```
SAU KHI CÓ DEPENDENCY MAP, ADS CHO PHÉP:

1. Tạo Application Groups (Nhóm Ứng Dụng):
   ├── "E-Commerce Frontend" = Web-01, Web-02, Web-03, Load Balancer
   ├── "E-Commerce Backend" = App-01, App-02
   └── "E-Commerce Data" = MySQL Primary, MySQL Replica, Redis

2. Gán 7R strategy cho từng nhóm:
   ├── "E-Commerce Frontend" → Rehost (MGN)
   ├── "E-Commerce Backend" → Replatform (EC2 → ECS containers)
   └── "E-Commerce Data" → Replatform (MySQL → Amazon Aurora)

3. Export groups sang Migration Hub để tracking
```

---

## 🔗 Migration Hub Integration — Tích Hợp Migration Hub

```
LUỒNG DỮ LIỆU ADS → MIGRATION HUB:

  AWS Application Discovery Service
  ├── Server inventory data
  ├── Performance metrics
  ├── Network connections
  └── Application groups
              │
              │ (tự động sync hoặc manual export)
              ▼
  AWS Migration Hub
  ├── Application Portfolio View (xem danh mục ứng dụng)
  │   └── Mỗi application group = 1 entry trong Hub
  ├── Strategy Assignment (gán chiến lược 7R)
  ├── Migration tracking (theo dõi tiến độ)
  └── Integration với MGN, DMS, SMS (theo dõi tiến trình migration)

CÁC TÍCH HỢP THÊM:
├── Migration Evaluator: Import ADS data → tính TCO
├── AWS MGN: Dùng ADS server list để chọn server cần rehost
└── AWS Partner tools: Nhiều partner có thể đọc ADS data
```

---

## 🛠️ Quy Trình Triển Khai Thực Tế

### Timeline Điển Hình (Thời Gian Điển Hình)

```
TUẦN 1: SETUP & INITIAL COLLECTION (Cài Đặt & Thu Thập Ban Đầu)

Ngày 1-2: Cài đặt
├── Cấu hình IAM user/role với ADS permissions
├── Triển khai Agentless Collector lên vSphere
└── Xác nhận dữ liệu đầu tiên về ADS Console

Ngày 3-7: Cài agent cho tier-1 apps
├── Cài Discovery Agent lên các app servers quan trọng
├── Verify agent đang gửi dữ liệu
└── Bắt đầu thấy network connections trong Console

TUẦN 2-4: DATA COLLECTION (Thu Thập Dữ Liệu)

├── Agentless Collector chạy tự động 24/7
├── Agent gửi dữ liệu mỗi 15 phút
├── Monitoring: kiểm tra collector/agent health hàng ngày
└── Chú ý: Cần bao phủ ít nhất 1 chu kỳ business (ví dụ: end-of-month
    processing — xử lý cuối tháng thường tốn nhiều tài nguyên hơn)

TUẦN 5: ANALYSIS & GROUPING (Phân Tích & Nhóm)

├── Export server inventory → xem xét, làm sạch dữ liệu
├── Review dependency map → xác nhận với application owners (chủ ứng dụng)
├── Tạo Application Groups trong ADS Console
├── Gán chiến lược 7R cho từng nhóm
└── Export sang Migration Hub và Migration Evaluator
```

---

## 📤 Export Dữ Liệu Và Phân Tích

### Export Formats (Định Dạng Xuất)

```
ADS HỖ TRỢ EXPORT:

1. CSV Export từ ADS Console:
   ├── Server inventory (tất cả servers)
   ├── Process list per server
   ├── Network connections
   └── Performance summary

2. Amazon S3 Export:
   ├── Continuous export (xuất liên tục) sang S3 bucket
   ├── Format: CSV files phân theo ngày
   └── Cho phép query bằng Amazon Athena (công cụ truy vấn dữ liệu lớn)

3. Migration Hub sync:
   └── Tự động hoặc manual push application groups

4. Migration Evaluator import:
   └── ADS data được tự động nhận dạng khi chạy Migration Evaluator
```

### Athena Query Phân Tích Nâng Cao

```sql
-- Ví dụ: Tìm các server có CPU utilization cao (> 80%)
-- → Cần right-size lên instance lớn hơn trên AWS

SELECT
    hostName,
    osName,
    cpuCount,
    totalRamInMB,
    AVG(cpuUsagePct) as avg_cpu_pct,
    MAX(cpuUsagePct) as peak_cpu_pct,
    AVG(ramUsageInMB) as avg_ram_mb
FROM "AwsDiscovery"."server_detail"
WHERE collectionDate > DATE '2026-05-01'
GROUP BY hostName, osName, cpuCount, totalRamInMB
HAVING AVG(cpuUsagePct) > 80
ORDER BY avg_cpu_pct DESC;

-- Ví dụ: Tìm tất cả kết nối từ server X đến server Y
-- → Xác nhận dependency

SELECT DISTINCT
    sourceIpAddress,
    destinationIpAddress,
    destinationPort,
    transportProtocol
FROM "AwsDiscovery"."network_connection"
WHERE sourceIpAddress = '192.168.1.20'
ORDER BY destinationPort;
```

---

## 🔒 Bảo Mật Và Quyền Riêng Tư

### Dữ Liệu Được Bảo Vệ Như Thế Nào

```
BẢO MẬT TRONG ADS:

Truyền tải (In-Transit):
├── Agent → ADS: HTTPS (TLS 1.2+)
└── Agentless Collector → ADS: HTTPS (TLS 1.2+)

Lưu trữ (At-Rest):
├── Dữ liệu trong ADS được mã hóa bằng AWS KMS (Key Management Service)
└── Data tồn tại trong region bạn chọn

Dữ liệu KHÔNG được thu thập (Privacy):
├── ❌ Nội dung file (file contents)
├── ❌ Database records (nội dung bản ghi DB)
├── ❌ Application data (dữ liệu ứng dụng)
├── ❌ Password hoặc credentials
└── ✅ Chỉ thu thập metadata (thông tin mô tả) và performance

IAM Permissions Cần Thiết:
├── ADS agentless collector: quyền write vào ADS
├── ADS agent: quyền write vào ADS
└── ADS console user: quyền read để xem data

Chính Sách IAM Tối Thiểu:
```json
{
  "Effect": "Allow",
  "Action": [
    "discovery:StartDataCollectionByAgentIds",
    "discovery:StopDataCollectionByAgentIds",
    "discovery:DescribeAgents",
    "discovery:DescribeConfigurations",
    "discovery:ExportConfigurations"
  ],
  "Resource": "*"
}
```

---

## ⚠️ Hạn Chế Và Lưu Ý

### Giới Hạn Dịch Vụ

```
LIMITS (Giới Hạn) CẦN BIẾT:

Agentless Collector:
├── Chỉ hỗ trợ VMware vSphere (không hỗ trợ Hyper-V, KVM native)
├── Cần network connectivity từ Collector → vCenter (TCP 443)
└── Cần network connectivity từ Collector → AWS (TCP 443)

Discovery Agent:
├── Số agents: mặc định 250 per account (có thể tăng qua Support)
├── Không hỗ trợ: AIX, Solaris, các Unix cũ
└── Windows XP / Server 2003 không được hỗ trợ

Retention (Thời Gian Lưu Dữ Liệu):
└── Data được giữ 90 ngày; export ra S3 nếu cần lưu lâu hơn

Pricing (Giá):
├── ADS Discovery itself: MIỄN PHÍ
└── Chỉ trả tiền cho S3 storage nếu dùng continuous export
```

### Những Điều Cần Lưu Ý Trong Thực Tế

```
⚠️ PITFALLS (CẠM BẪY PHỔ BIẾN):

1. Thu thập quá ngắn → dữ liệu không đại diện:
   Problem: Chạy 3 ngày → bỏ sót peak load cuối tháng
   Solution: Chạy tối thiểu 4 tuần, lý tưởng 6-8 tuần

2. Chỉ dùng Agentless → dependency map không đầy đủ:
   Problem: Không thấy được process-level connections
   Solution: Cài Agent trên tier-1 applications

3. Không validate dependency map với application team:
   Problem: ADS chỉ thấy network connections, không biết ý nghĩa
   Solution: Review map với developers để xác nhận và bổ sung

4. Bỏ qua servers đã tắt (powered off):
   Problem: Agentless Collector chỉ collect running VMs
   Solution: Kiểm tra vCenter để thêm VMs đang tắt vào inventory thủ công

5. Không tính external dependencies (phụ thuộc bên ngoài):
   Problem: App kết nối đến SaaS API, on-premises ERP của đối tác
   Solution: Kiểm tra network flows để tìm kết nối ra internet
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: AWS Application Discovery Service là gì và tại sao cần nó?**

> ADS là dịch vụ tự động khám phá (discover) hạ tầng on-premises — thu thập thông tin về máy chủ, ứng dụng, hiệu năng và kết nối mạng giữa các thành phần. Cần nó vì:
>
> 1. **Inventory chính xác**: Tránh bỏ sót server khi lên kế hoạch migration
> 2. **Dependency mapping**: Biết ứng dụng nào phụ thuộc vào ứng dụng nào → di chuyển đúng thứ tự, tránh downtime ngoài dự kiến
> 3. **Right-sizing data**: Dữ liệu utilization thực tế → chọn EC2 instance size phù hợp → tiết kiệm chi phí
> 4. **Input cho Migration Evaluator**: Tính TCO chính xác dựa trên dữ liệu thực

---

**Q: Phân biệt Agentless Collector và Discovery Agent trong ADS?**

> - **Agentless Collector**: Là VM OVA cài trên VMware vSphere, đọc data qua vCenter API. Thu thập metadata và performance tổng quát. Triển khai nhanh, không cần quyền OS. Phù hợp VMware environments.
>
> - **Discovery Agent**: Phần mềm cài trực tiếp lên từng server (Windows/Linux). Thu thập dữ liệu sâu hơn: running processes, network connections chi tiết (IP:port pairs), disk I/O. Cho phép vẽ dependency map chính xác hơn ở mức process.
>
> Thực tế: Dùng cả hai — Agentless cho inventory nhanh toàn bộ, Agent cho các ứng dụng quan trọng cần phân tích dependency kỹ.

---

**Q: Tại sao phải chạy ADS ít nhất 2-4 tuần?**

> Để thu thập đủ mẫu dữ liệu hiệu năng đại diện:
> - Bao gồm cả **weekday và weekend patterns** (pattern ngày thường và cuối tuần)
> - Bao gồm **peak periods** — ví dụ xử lý cuối tháng, batch jobs ban đêm
> - Đủ để tính **P95 utilization** thay vì chỉ average — tránh undersizing
>
> Nếu chỉ chạy 3 ngày, có thể bỏ sót peak load → right-sizing sai → EC2 bị thiếu tài nguyên sau khi migrate.

---

### Câu Hỏi Nâng Cao

**Q: Bạn có môi trường Hyper-V, không có VMware. Làm thế nào để dùng ADS?**

> Agentless Collector chỉ hỗ trợ VMware vSphere. Với Hyper-V có hai lựa chọn:
> 1. **Discovery Agent**: Cài trực tiếp lên từng Windows Server VM. Cách tiếp cận này thực ra cho dữ liệu **sâu hơn** Agentless Collector vì thu thập được process-level data.
> 2. **PowerShell export**: Dùng PowerShell để export VM inventory từ Hyper-V Manager, sau đó import thủ công vào Migration Evaluator qua template Excel.

---

**Q: Làm thế nào để handle (xử lý) server không thể cài Discovery Agent (ví dụ: vendor-managed appliance)?**

> Ba cách tiếp cận:
> 1. **Agentless** nếu là VMware VM: Vẫn lấy được VM-level config và basic metrics
> 2. **Network-based discovery**: Phân tích network flows từ các server xung quanh — thấy ai kết nối vào, vào port nào
> 3. **Manual documentation**: Yêu cầu vendor cung cấp specs, sau đó nhập thủ công vào Migration Evaluator
>
> Quan trọng: Ghi chú rõ trong inventory rằng dữ liệu của server này là manual/estimated, không phải từ ADS — để team biết mức độ tin cậy.

---

**Q: Sau khi có dependency map từ ADS, bạn làm gì tiếp theo?**

> 5 bước:
> 1. **Validate** (xác nhận) với application owners — ADS chỉ thấy network connections, developers biết ý nghĩa thực sự
> 2. **Bổ sung** external dependencies ADS không thấy (SaaS services, external APIs)
> 3. **Tạo Application Groups** trong ADS Console — nhóm các servers cùng thuộc một ứng dụng
> 4. **Gán 7R strategy** cho mỗi group — Retire/Retain/Rehost/Relocate/Repurchase/Replatform/Refactor
> 5. **Lập Migration Wave Plan** — thứ tự di chuyển từ "leaf nodes" (không có dependency) đến "root nodes" (nhiều thứ phụ thuộc vào)

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 07-discovery-assessment
