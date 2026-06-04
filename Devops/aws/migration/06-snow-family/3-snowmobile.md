# AWS Snowmobile — Xe Tải Dữ Liệu 100 PB Cho Exabyte Migration

> **AWS Snowmobile** là giải pháp di chuyển dữ liệu cực lớn duy nhất trên thế giới theo đúng nghĩa đen: một **container 45 feet** (khoảng 13.7 m) được kéo bởi xe tải bán tải, có thể chứa đến **100 petabyte** (100 PB = 100,000 TB) dữ liệu mỗi chuyến. Snowmobile giải quyết bài toán di chuyển exabyte-scale (hàng exabyte = hàng triệu TB) — ví dụ khi gộp nhiều datacenter lớn vào AWS, hoặc di chuyển kho lưu trữ media khổng lồ của hãng phim, đài truyền hình.

## 📚 Mục Lục (Table of Contents)

1. [Snowmobile Là Gì? Tại Sao Cần?](#snowmobile-là-gì-tại-sao-cần)
2. [Thông Số Kỹ Thuật](#thông-số-kỹ-thuật)
3. [Quy Trình Triển Khai Snowmobile](#quy-trình-triển-khai-snowmobile)
4. [Bảo Mật — Cấp Độ Cao Nhất](#bảo-mật--cấp-độ-cao-nhất)
5. [Network Connection Tại Datacenter](#network-connection-tại-datacenter)
6. [Tính Toán Thời Gian Và Chi Phí](#tính-toán-thời-gian-và-chi-phí)
7. [So Sánh Với Nhiều Snowball Edge](#so-sánh-với-nhiều-snowball-edge)
8. [Các Trường Hợp Sử Dụng Thực Tế](#các-trường-hợp-sử-dụng-thực-tế)
9. [Hạn Chế Của Snowmobile](#hạn-chế-của-snowmobile)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🚛 Snowmobile Là Gì? Tại Sao Cần?

### Bài Toán Exabyte-Scale Migration

```
Tình huống thực tế:
Một tổ chức tài chính lớn quyết định đóng cửa 3 datacenter
và chuyển toàn bộ 50 PB dữ liệu lịch sử lên AWS S3.

Kết nối tốt nhất: 10 Gbps dedicated line (Direct Connect)

Tính toán:
├── 50 PB = 50,000 TB = 50,000,000 GB
├── 10 Gbps = 1.25 GB/s (lý thuyết)
├── Thực tế hiệu quả: ~60% = 0.75 GB/s
├── Thời gian: 50,000,000 GB ÷ 0.75 GB/s = 66.7M giây
└── = 772 ngày ≈ 2 năm và 1 tháng!

Chi phí data transfer (egress from on-premises qua Direct Connect):
└── 50 PB × $0.02/GB = ~$1 triệu USD chỉ phí vận chuyển!

→ Không khả thi về mặt thời gian và chi phí
→ Giải pháp: AWS Snowmobile
```

### Snowmobile — Giải Pháp Vật Lý Cho Vấn Đề Số

```
AWS Snowmobile = Xe tải container 45 feet với:
├── 100 PB storage capacity (100 petabyte)
├── Multiple 40 GbE network interfaces
│   (tổng throughput lên đến 1 Tbps — 1 terabit/giây)
├── Hệ thống điện dự phòng và làm mát riêng biệt
├── Bảo vệ vật lý cấp cao nhất (GPS, CCTV, bảo vệ vũ trang)
└── AWS chịu trách nhiệm vận hành toàn bộ

Thời gian tương tự ví dụ 50 PB trên:
├── Copy tốc độ 1 Tbps → 50 PB mất ~5 ngày
├── Vận chuyển đến AWS facility: 2-5 ngày
├── AWS import vào S3: 5-10 ngày
└── Tổng: ~2-3 tuần thay vì 2 năm!
```

---

## 📐 Thông Số Kỹ Thuật

```
┌─────────────────────────────────────────────────────────────────┐
│                  AWS SNOWMOBILE — SPECS                         │
├─────────────────────────────┬───────────────────────────────────┤
│ Hình dạng vật lý            │ Shipping container 45 feet        │
│                             │ (≈ 13.7 m × 2.4 m × 2.6 m)       │
│ Phương tiện kéo             │ Xe tải bán tải (semi-truck)       │
├─────────────────────────────┼───────────────────────────────────┤
│ Dung lượng tối đa           │ 100 PB (petabyte) = 100,000 TB    │
├─────────────────────────────┼───────────────────────────────────┤
│ Network Connectivity        │ Nhiều × 40 GbE interface          │
│ Tổng throughput             │ Lên đến 1 Tbps (terabit/giây)    │
├─────────────────────────────┼───────────────────────────────────┤
│ Nguồn điện                  │ Kết nối vào lưới điện datacenter  │
│                             │ + UPS và generator dự phòng       │
│ Làm mát                     │ Hệ thống cooling (làm lạnh) riêng │
├─────────────────────────────┼───────────────────────────────────┤
│ GPS Tracking                │ ✅ 24/7                           │
│ CCTV                        │ ✅ Camera quan sát liên tục        │
│ Bảo vệ vật lý               │ ✅ Nhân viên bảo vệ vũ trang      │
│ Cảm biến xâm nhập           │ ✅ Alarm phát hiện mở container   │
├─────────────────────────────┼───────────────────────────────────┤
│ Mã hóa                      │ AES-256 + KMS                     │
│ Tính năng compute           │ Không có (chỉ storage)            │
│ Availability                │ Một số AWS regions lớn            │
└─────────────────────────────┴───────────────────────────────────┘
```

### Bối Cảnh Dung Lượng 100 PB

```
100 PB = 100 Petabyte = 100,000 Terabyte = 100,000,000 Gigabyte

Để hình dung:
├── Toàn bộ text trên Wikipedia (tất cả ngôn ngữ): ~20 GB → 100 PB ÷ 20 GB = 5 triệu bộ Wikipedia
├── Một bộ phim HD (Blu-ray): ~50 GB → 2 triệu bộ phim
├── Một bài nhạc MP3 (~5 MB): 20 tỷ bài nhạc
├── Genome con người (~3 GB): 33 triệu bộ genome
└── Ảnh selfie (~5 MB): 20 tỷ ảnh

Đây là quy mô của:
├── Kho lưu trữ video của một mạng truyền hình lớn (20+ năm nội dung)
├── Toàn bộ dữ liệu của một ngân hàng lớn (30+ năm giao dịch)
└── Hàng nghìn petabyte = Khi cần nhiều hơn → nhiều xe Snowmobile
```

---

## 🔄 Quy Trình Triển Khai Snowmobile

### Quy Trình 6 Bước Chi Tiết

```
BƯỚC 1: LIÊN HỆ AWS VÀ LÊN KẾ HOẠCH
┌─────────────────────────────────────────────────────────────────┐
│  Không đặt hàng qua Console như Snowball — cần liên hệ AWS      │
│                                                                  │
│  AWS Solutions Architect đánh giá:                               │
│  ├── Tổng dữ liệu cần di chuyển                                  │
│  ├── Vị trí datacenter (có thể đậu xe không?)                   │
│  ├── Cơ sở hạ tầng điện (cần bao nhiêu kW?)                     │
│  ├── Network bandwidth tại datacenter (10/40 GbE?)              │
│  └── Timeline (thời gian biểu) mong muốn                        │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 2: AWS ĐƯA XE ĐẾN DATACENTER CỦA BẠN
┌─────────────────────────────────────────────────────────────────┐
│  AWS lái xe Snowmobile đến địa điểm của bạn                      │
│  Kỹ thuật viên AWS đi kèm để thiết lập                           │
│  Kiểm tra điều kiện môi trường: điện, mạng, không gian đậu xe   │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 3: KẾT NỐI VÀ THIẾT LẬP
┌─────────────────────────────────────────────────────────────────┐
│  Kỹ thuật viên AWS kết nối:                                       │
│  ├── Nguồn điện: cáp điện lớn từ UPS của datacenter             │
│  ├── Mạng: Multiple 40 GbE fibre cables vào switch              │
│  └── Kiểm tra kết nối và bảo mật                                 │
│                                                                  │
│  Bạn nhận:                                                       │
│  └── Credentials để truy cập Snowmobile như S3 endpoint         │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 4: COPY DỮ LIỆU
┌─────────────────────────────────────────────────────────────────┐
│  Snowmobile xuất hiện như một S3 endpoint trên mạng nội bộ      │
│                                                                  │
│  Phương thức copy:                                               │
│  ├── AWS DataSync (đối tác tốt nhất cho Snowmobile)             │
│  ├── S3-compatible API từ mọi tool: rclone, s3cmd               │
│  └── Phần mềm backup: Veeam, Commvault, Veritas                  │
│                                                                  │
│  Tốc độ tối đa: 1 Tbps = 125 GB/s                               │
│  100 PB tại 125 GB/s = ~9.3 ngày (lý thuyết)                   │
│  Thực tế: 2-4 tuần tùy số lượng máy chủ copy                   │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 5: AWS LÁI XE VỀ AWS FACILITY
┌─────────────────────────────────────────────────────────────────┐
│  Kỹ thuật viên ngắt kết nối và chuẩn bị xe lên đường           │
│  GPS tracking: bạn theo dõi vị trí xe real-time                 │
│  Bảo vệ vũ trang đi kèm suốt hành trình                        │
│  AWS không bao giờ để xe không có người giám sát                │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 6: IMPORT VÀO S3
┌─────────────────────────────────────────────────────────────────┐
│  Tại AWS facility:                                               │
│  ├── Giải mã dữ liệu bằng KMS key của bạn                       │
│  ├── Import vào S3 buckets đã chỉ định                           │
│  └── Verify (xác minh) checksum toàn bộ dữ liệu                 │
│                                                                  │
│  Thời gian import: 1-3 tuần tùy dung lượng                      │
│  Thông báo: Email + SNS khi từng batch hoàn tất                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Bảo Mật — Cấp Độ Cao Nhất

### Bảo Mật Vật Lý

```
Snowmobile có cấp độ bảo mật vật lý cao nhất trong Snow Family:

GPS TRACKING (Theo Dõi GPS):
├── Hệ thống GPS 24/7 — AWS biết vị trí xe chính xác mọi lúc
├── Geofencing (hàng rào địa lý): cảnh báo nếu xe ra khỏi lộ trình
└── Bạn cũng có thể theo dõi trạng thái qua AWS Console

VIDEO SURVEILLANCE (Giám Sát Qua Camera):
├── CCTV (Closed-Circuit Television — Camera Quan Sát) bên ngoài xe
├── Camera bên trong container
└── Ghi lại liên tục, lưu trữ và monitored bởi AWS Security

ARMED ESCORT (Bảo Vệ Vũ Trang):
├── Nhân viên bảo vệ vũ trang đi kèm khi vận chuyển
└── AWS không công bố chi tiết quy trình bảo vệ vì lý do an ninh

PHYSICAL ALARM (Cảnh Báo Vật Lý):
└── Cảm biến rung động, nhiệt độ, ánh sáng (phát hiện mở container)
    Nếu container bị mở trái phép → hệ thống cảnh báo và khóa dữ liệu
```

### Bảo Mật Dữ Liệu

```
MÃ HÓA:
├── AES-256 (256-bit Advanced Encryption Standard)
├── Mỗi ổ đĩa trong Snowmobile được mã hóa độc lập
├── Khóa mã hóa lưu trong AWS KMS — không bao giờ lưu trên xe
└── CMK (Customer Master Key — Khóa Chính Khách Hàng): bạn kiểm soát hoàn toàn

NETWORK SECURITY (Bảo Mật Mạng):
├── Kết nối mã hóa giữa máy chủ của bạn và Snowmobile
├── Dedicated VLAN (Virtual Local Area Network) riêng biệt
└── Firewall (tường lửa) cứng trên thiết bị

SAU KHI IMPORT:
└── AWS thực hiện secure erase NIST 800-88 trên tất cả ổ đĩa
    Certificate of Data Destruction (Chứng chỉ Hủy Dữ Liệu) nếu cần

TUÂN THỦ (Compliance):
├── HIPAA (Y tế)
├── PCI-DSS (Tài chính)
├── SOC 1/2/3
├── FedRAMP (Chính phủ Mỹ)
└── ISO 27001
```

---

## 🌐 Network Connection Tại Datacenter

### Yêu Cầu Hạ Tầng Mạng

```
Để tối đa hóa tốc độ copy, datacenter của bạn cần:

NETWORK INTERFACES:
├── Tối thiểu: 10 GbE kết nối vào Snowmobile (1.25 GB/s)
├── Tốt hơn: 40 GbE (5 GB/s)
└── Tốt nhất: Nhiều × 40 GbE (gần 1 Tbps)

SWITCH INFRASTRUCTURE (Cơ Sở Hạ Tầng Switch Mạng):
├── Core switch hỗ trợ 40 GbE hoặc 100 GbE
└── Patch panel (tủ đấu nối) để kết nối cable từ xe vào switch

POWER REQUIREMENTS (Yêu Cầu Điện):
├── Snowmobile tiêu thụ điện lớn
├── Cần 208V 3-phase power hoặc tương đương
└── AWS kỹ thuật viên đánh giá và xác nhận trước khi triển khai

PHYSICAL SPACE (Không Gian Vật Lý):
├── Bãi đậu xe bán tải gần cổng datacenter
├── Khoảng cách từ xe vào trong datacenter (dài cable hạn chế)
└── Đường vào phải đủ rộng cho xe tải lớn

VÍ DỤ THIẾT LẬP:

[Datacenter Servers]
        │ 
        │ (nhiều × 10 GbE hoặc 40 GbE)
        ▼
[Core Switch trong Datacenter]
        │
        │ (fiber optic cable kéo ra cửa)
        ▼
[Snowmobile Network Interfaces]
        │ (kết nối nội bộ)
        ▼
[100 PB Storage Arrays bên trong container]
```

### Chiến Lược Copy Tối Ưu

```
Để đạt throughput (thông lượng) tối đa:

1. PARALLEL SOURCE SERVERS (Nhiều Máy Chủ Nguồn):
   Thay vì 1 server copy → 10-50 servers copy song song
   Mỗi server có 10 GbE → 50 servers × 10 Gbps = 500 Gbps tổng

2. STREAMING READS (Đọc Tuần Tự):
   Sequential reads nhanh hơn random reads trên HDD/SSD
   Sắp xếp dữ liệu để copy sequential thay vì random

3. DATASYNC AGENT ĐỘI NGŨ:
   ├── Cài DataSync agents trên nhiều servers
   ├── Mỗi agent quản lý một phần của dữ liệu
   └── DataSync tự động tối ưu parallel transfers

4. TRÁNH SMALL FILES:
   └── Nhiều file nhỏ gây overhead cao
       Gộp thành tar hoặc zip trước khi copy nếu có thể
```

---

## 📊 Tính Toán Thời Gian Và Chi Phí

### Thời Gian Migration Ước Tính

```
100 PB qua các phương thức khác nhau:

PHƯƠNG THỨC 1: Internet (1 Gbps)
├── Thời gian lý thuyết: 100 PB × 10^6 GB ÷ 0.125 GB/s
│   = 800,000,000 giây = 25 năm
└── → Hoàn toàn không khả thi

PHƯƠNG THỨC 2: Direct Connect (10 Gbps)
├── 100 PB ÷ 1.25 GB/s = 80,000,000 giây = 2.5 năm
└── → Không khả thi về thời gian

PHƯƠNG THỨC 3: Direct Connect (100 Gbps)
├── 100 PB ÷ 12.5 GB/s = 8,000,000 giây = 92 ngày
└── → Khả thi về thời gian nhưng chi phí rất cao

PHƯƠNG THỨC 4: Snowmobile (1 Tbps = 1000 Gbps)
├── Copy: 100 PB ÷ 125 GB/s ≈ 9 ngày (lý thuyết)
├── Thực tế: 2-4 tuần (tùy số servers copy song song)
├── Vận chuyển + import: 1-2 tuần
└── TỔNG: 3-6 tuần ✅

PHƯƠNG THỨC 5: Nhiều Snowball Edge (15 nodes × 80 TB = 1.2 PB/lần)
├── 100 PB ÷ 1.2 PB/batch = ~84 batches
├── Mỗi batch: 1-2 tuần
└── TỔNG: 84 × 1.5 tuần = ~2.5 năm → Không thực tế
    (Trừ khi có hàng trăm Snowball Edge song song)
```

### Chi Phí So Sánh

```
CHI PHÍ SNOWMOBILE (ước tính):
├── Phí thiết bị: Không công bố (liên hệ AWS để báo giá)
├── Data transfer vào S3: MIỄN PHÍ
└── Chi phí thực tế thấp hơn NHIỀU so với Direct Connect

CHI PHÍ DIRECT CONNECT (100 Gbps, 3 tháng):
├── Direct Connect port (100 Gbps): ~$22,000/tháng
├── 3 tháng: $66,000
├── + Data transfer egress fees nếu có
└── Tổng: $70,000 - $100,000+

CHI PHÍ INTERNET UPLOAD (100 PB):
├── Data transfer out từ datacenter: rất lớn
└── Không khả thi về kỹ thuật (2.5 năm)

→ Snowmobile thường có tổng chi phí THẤP HƠN cho > 10 PB
  ngay cả khi tính phí thiết bị và vận chuyển
```

---

## ⚖️ So Sánh Với Nhiều Snowball Edge

### Khi Nào Dùng Snowmobile vs Nhiều Snowball Edge?

```
┌─────────────────────────────────────────────────────────────────────┐
│         SNOWMOBILE vs NHIỀU SNOWBALL EDGE                           │
├────────────────────────┬──────────────────┬─────────────────────────┤
│ Tiêu Chí               │ Snowmobile       │ Nhiều Snowball Edge     │
├────────────────────────┼──────────────────┼─────────────────────────┤
│ Dung lượng             │ 100 PB/xe        │ 80 TB/thiết bị          │
│ Tốc độ copy            │ Lên đến 1 Tbps   │ ~10 Gbps/thiết bị       │
│ Edge computing         │ Không            │ ✅ (cả hai phiên bản)   │
│ Tính linh hoạt         │ Thấp hơn         │ Cao hơn                 │
│ Yêu cầu hạ tầng        │ Cao (điện, mạng) │ Thấp (cắm vào LAN)     │
│ Số lần kế hoạch        │ Một lần lớn      │ Nhiều lần nhỏ           │
│ Phù hợp với            │ > 10 PB          │ < 10 PB                 │
│ Compute tại biên       │ Không            │ ✅                      │
│ Deployment time        │ Tuần để chuẩn bị │ Ngày                   │
├────────────────────────┼──────────────────┼─────────────────────────┤
│ Ngưỡng quyết định      │                  │                         │
│ Dùng Snowmobile khi:   │ > 10 PB          │ ≤ 10 PB                 │
│                        │ Một datacenter   │ Nhiều địa điểm          │
│                        │ Cần tốc độ cao   │ Cần tính linh hoạt      │
└────────────────────────┴──────────────────┴─────────────────────────┘
```

### Công Thức Quyết Định

```
Dữ liệu cần di chuyển: X PB

Nếu X ≥ 10 PB VÀ dữ liệu tập trung một nơi:
└── Snowmobile (1 hoặc nhiều xe)

Nếu X < 10 PB HOẶC dữ liệu phân tán nhiều địa điểm:
└── Nhiều Snowball Edge (linh hoạt hơn, dễ logistics hơn)

Nếu cần edge computing sau migration:
└── Snowball Edge (Snowmobile không có compute)

Nếu X > 100 PB:
└── Nhiều Snowmobile song song (AWS hỗ trợ nhiều xe cùng lúc)
```

---

## 🏢 Các Trường Hợp Sử Dụng Thực Tế

### Case 1: Datacenter Consolidation (Gộp Nhiều Datacenter)

```
Tình huống:
Ngân hàng lớn quyết định đóng cửa 5 datacenter khu vực
Tổng dữ liệu: 80 PB (dữ liệu giao dịch 30 năm, backup, archive)
Timeline: Hoàn thành trong 6 tháng

Chiến lược:
├── 1 Snowmobile cho mỗi datacenter lớn nhất (3 xe)
├── Snowball Edge cho 2 datacenter nhỏ hơn
├── DMS (Database Migration Service) cho databases đang chạy
└── MGN (Application Migration Service) cho application servers

Kết quả:
├── 3 xe Snowmobile: 3 × 25 PB = 75 PB
├── 5 Snowball Edge: 5 × 80 TB = 400 TB còn lại
└── Timeline: 3 tháng (nhanh hơn kế hoạch)
```

### Case 2: Media & Entertainment Archive (Kho Lưu Trữ Media)

```
Tình huống:
Studio phim lớn có 50 PB phim raw footage (cảnh quay thô)
Muốn chuyển lên S3 và S3 Glacier để tiết kiệm chi phí lưu trữ

Vấn đề:
├── Tape library (thư viện băng từ) không có giao diện network tốt
├── Dữ liệu phân tán nhiều format và codec khác nhau
└── Cần giữ nguyên metadata và directory structure

Giải pháp:
├── Snowmobile × 1: 50 PB raw footage
├── DataSync agent: chuyển đổi metadata
├── S3 lifecycle policy: phim < 5 năm ở S3 Standard
│                        phim > 5 năm xuống S3 Glacier Deep Archive
└── Tiết kiệm chi phí: ~70% so với on-premises tape

Kết quả:
├── Migration hoàn tất trong 6 tuần
└── Chi phí lưu trữ giảm từ $2M/năm xuống $600K/năm
```

### Case 3: Scientific Data Repository (Kho Dữ Liệu Khoa Học)

```
Tình huống:
Viện nghiên cứu thiên văn học có 200 PB dữ liệu từ kính viễn vọng
Muốn chia sẻ với cộng đồng khoa học toàn cầu qua AWS

Giải pháp:
├── 2 × Snowmobile (2 × 100 PB = 200 PB)
├── Import vào S3 → Public dataset on AWS
├── Requester Pays (người dùng trả phí download)
└── Amazon Athena để query trực tiếp trên S3

Kết quả:
└── Data available to 10,000 researchers worldwide
    mà không cần duy trì datacenter riêng
```

### Case 4: Government/Military Migration

```
Tình huống:
Cơ quan chính phủ cần di chuyển hồ sơ 50 năm lên AWS GovCloud
Tổng: 20 PB tài liệu đã số hóa

Yêu cầu đặc biệt:
├── FedRAMP High authorization (ủy quyền bảo mật cao nhất)
├── Không được truyền qua internet công cộng
└── Chain of custody (chuỗi kiểm soát) dữ liệu được ghi lại

Giải pháp:
├── Snowmobile vào AWS GovCloud region
├── Background check (kiểm tra lý lịch) cho tất cả nhân viên AWS tham gia
└── Thủ tục audit (kiểm toán) đặc biệt theo quy định liên bang
```

---

## ⚠️ Hạn Chế Của Snowmobile

```
Snowmobile KHÔNG phù hợp cho:

1. DỮ LIỆU PHÂN TÁN NHIỀU ĐỊA ĐIỂM:
   └── Xe tải không thể đến 50 chi nhánh khác nhau
       → Dùng Snowball Edge ở mỗi địa điểm

2. EDGE COMPUTING:
   └── Snowmobile không có CPU để chạy ứng dụng
       → Dùng Snowball Edge Compute Optimized

3. DỮ LIỆU < 10 PB:
   └── Chi phí logistics của Snowmobile không justify
       → Dùng Snowball Edge hoặc DataSync

4. CẦN EXPORT (NHẬN DỮ LIỆU TỪ S3):
   └── Snowmobile chỉ hỗ trợ import vào S3
       (Export hiện không được hỗ trợ)

5. VÙNG KHÔNG CÓ SNOWMOBILE:
   └── Chỉ available tại một số AWS regions lớn
       Hỏi AWS sales team trước khi lên kế hoạch

6. CẦN TÍNH TOÁN TẠI CHỖ (OFFLINE PROCESSING):
   └── Snowmobile chỉ là storage, không chạy compute
       → Kết hợp với Snowball Edge nếu cần xử lý trước khi ship

7. TIMELINE RẤT GẤP (< 2 TUẦN TỔNG):
   └── Chuẩn bị Snowmobile (logistics, hợp đồng) mất 2-4 tuần
       → Với dữ liệu nhỏ hơn: Snowball Edge đặt hàng qua Console ngay
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q1: AWS Snowmobile là gì và khi nào dùng?**

> Snowmobile là container 45 feet chứa 100 PB storage, kéo bởi xe tải bán tải. AWS giao xe đến datacenter của bạn, kết nối vào mạng, bạn copy dữ liệu, AWS lái xe về và import vào S3. Dùng khi cần di chuyển > 10 PB và dữ liệu tập trung tại một hoặc vài datacenter. Ví dụ: datacenter consolidation, media archive migration, scientific data repository.

**Q2: Tại sao cần Snowmobile thay vì Direct Connect tốc độ cao?**

> Với 100 PB và đường 100 Gbps:
> - Thời gian transfer lý thuyết: 100 PB ÷ 12.5 GB/s ≈ 92 ngày
> - Chi phí Direct Connect 100 Gbps: ~$22,000/tháng × 3 tháng = $66,000+
> - Snowmobile: copy tốc độ 1 Tbps → ~2-4 tuần, chi phí thấp hơn nhiều
>
> Snowmobile nhanh hơn 10x và rẻ hơn cho exabyte-scale.

**Q3: Snowmobile bảo mật dữ liệu như thế nào trong quá trình vận chuyển?**

> Nhiều lớp bảo mật:
> 1. **Mã hóa**: AES-256, khóa trong KMS — AWS không thể đọc dữ liệu
> 2. **GPS tracking**: theo dõi vị trí 24/7
> 3. **CCTV**: camera giám sát liên tục trong và ngoài container
> 4. **Bảo vệ vũ trang**: đi kèm trong suốt hành trình
> 5. **Alarm**: cảm biến phát hiện mở container trái phép
> 6. **Tuân thủ**: HIPAA, PCI-DSS, FedRAMP, SOC 1/2/3

**Q4: Snowmobile khác Snowball Edge như thế nào?**

> | | Snowmobile | Snowball Edge |
> |---|---|---|
> | Dung lượng | 100 PB | 80 TB |
> | Compute | Không | ✅ |
> | Phù hợp | > 10 PB | < 10 PB |
> | Linh hoạt | Thấp (xe tải) | Cao (vali nhỏ) |
> | Triển khai | Tuần | Ngày |
>
> Chọn Snowball Edge cho dữ liệu < 10 PB hoặc cần edge computing.

**Q5: Khách hàng có 500 PB cần di chuyển. Giải pháp?**

> 500 PB ÷ 100 PB/xe = 5 Snowmobile. AWS hỗ trợ nhiều xe song song:
> - 5 xe cùng lúc → tổng throughput lên đến 5 Tbps
> - Chia datacenter thành 5 vùng, mỗi xe phụ trách một vùng
> - Timeline: ~4-6 tuần
>
> So sánh: Dùng Direct Connect 100 Gbps sẽ mất hơn 1 năm và chi phí hàng triệu USD.

**Q6: Nếu dữ liệu nằm ở 20 địa điểm khác nhau trên toàn quốc, có nên dùng Snowmobile không?**

> Không lý tưởng. Snowmobile phù hợp khi dữ liệu tập trung tại 1-3 datacenter lớn. Với 20 địa điểm phân tán:
> - Đặt Snowball Edge tại mỗi địa điểm (có thể đặt qua Console trong vài phút)
> - Hoặc dùng DataSync nếu bandwidth cho phép
> - Dùng Migration Hub để theo dõi tất cả các địa điểm song song

**Q7: Sau khi Snowmobile import xong, dữ liệu có thể dùng ngay không?**

> Có — dữ liệu được import vào S3, ngay lập tức có thể:
> - Truy cập từ EC2, Lambda, EMR, Athena
> - Thiết lập S3 lifecycle policies (chuyển sang Glacier nếu cần)
> - Dùng S3 Transfer Acceleration để phân phối toàn cầu
> - Cấu hình S3 Replication sang region khác cho DR

---

## 🗺️ Điều Hướng

- [README.md](./README.md) — Tổng quan & so sánh Snow Family
- [1-snowcone.md](./1-snowcone.md) — Snowcone: 8-14 TB, edge computing nhỏ gọn
- [2-snowball-edge.md](./2-snowball-edge.md) — Snowball Edge: Storage vs Compute Optimized

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Thuộc Module:** 06-snow-family (AWS Migration & Transfer Services)
