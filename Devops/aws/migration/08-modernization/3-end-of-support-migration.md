# AWS End-of-Support Migration — Di Chuyển Khỏi Phần Mềm Hết Hỗ Trợ

> **End-of-Support (EOS — Hết Hỗ Trợ)** xảy ra khi Microsoft dừng cung cấp security patches cho Windows Server hoặc SQL Server. Chạy phần mềm hết hỗ trợ đặt tổ chức vào rủi ro bảo mật nghiêm trọng và vi phạm compliance (tuân thủ). AWS cung cấp hai giải pháp: **Extended Security Updates (ESU — Cập Nhật Bảo Mật Mở Rộng) miễn phí trên EC2** và **End-of-Support Migration Program (EMP — Chương Trình Di Chuyển Hết Hỗ Trợ)** để nâng cấp ứng dụng lên phiên bản OS/SQL mới hơn.

## 📚 Mục Lục (Table of Contents)

1. [End-of-Support Là Gì?](#end-of-support-là-gì)
2. [Rủi Ro Khi Tiếp Tục Dùng EOS Software](#rủi-ro-khi-tiếp-tục-dùng-eos-software)
3. [Giải Pháp AWS: ESU Miễn Phí Trên EC2](#giải-pháp-aws-esu-miễn-phí-trên-ec2)
4. [End-of-Support Migration Program (EMP)](#end-of-support-migration-program-emp)
5. [Lộ Trình Upgrade Windows Server](#lộ-trình-upgrade-windows-server)
6. [Lộ Trình Upgrade SQL Server](#lộ-trình-upgrade-sql-server)
7. [So Sánh Các Phương Án](#so-sánh-các-phương-án)
8. [Compliance Và Audit](#compliance-và-audit)
9. [Checklist EOS Migration](#checklist-eos-migration)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ⏰ End-of-Support Là Gì?

### Vòng Đời Hỗ Trợ Microsoft

```
MICROSOFT SUPPORT LIFECYCLE (VÒNG ĐỜI HỖ TRỢ MICROSOFT):

Microsoft cung cấp 2 giai đoạn hỗ trợ:

1. Mainstream Support (Hỗ Trợ Chính):
   ├── Nhận: Security patches, bug fixes, new features
   ├── Thời gian: 5 năm sau khi phát hành
   └── Ví dụ: Windows Server 2016 mainstream hết: 2022-01-11

2. Extended Support (Hỗ Trợ Mở Rộng):
   ├── Nhận: CHỈ security patches (không có feature mới)
   ├── Thời gian: Thêm 5 năm sau Mainstream
   └── Ví dụ: Windows Server 2016 extended hết: 2027-01-12

3. End-of-Support (Hết Hỗ Trợ):
   ├── Không còn BẤT KỲ patch nào từ Microsoft
   ├── Mọi lỗ hổng mới phát hiện → KHÔNG được vá
   └── Đây là thời điểm nguy hiểm nhất
```

### Mốc Thời Gian Quan Trọng

```
CÁC PHIÊN BẢN ĐÃ HẾT HOẶC SẮP HẾT HỖ TRỢ:

WINDOWS SERVER:
┌────────────────────────────────┬────────────────────────┬───────────────────┐
│ Phiên Bản                      │ Ngày EOS               │ Trạng Thái        │
├────────────────────────────────┼────────────────────────┼───────────────────┤
│ Windows Server 2003/R2         │ 2015-07-14             │ ❌ HẾT 11 năm     │
│ Windows Server 2008/R2         │ 2020-01-14             │ ❌ HẾT 6 năm      │
│ Windows Server 2012/R2         │ 2023-10-10             │ ❌ HẾT 2023       │
│ Windows Server 2016            │ 2027-01-12             │ ⚠️ Còn ~1 năm     │
│ Windows Server 2019            │ 2029-01-09             │ ✅ Còn ~3 năm     │
│ Windows Server 2022            │ 2031-10-14             │ ✅ Còn ~5 năm     │
└────────────────────────────────┴────────────────────────┴───────────────────┘

SQL SERVER:
┌────────────────────────────────┬────────────────────────┬───────────────────┐
│ Phiên Bản                      │ Ngày EOS               │ Trạng Thái        │
├────────────────────────────────┼────────────────────────┼───────────────────┤
│ SQL Server 2008/R2             │ 2019-07-09             │ ❌ HẾT 7 năm      │
│ SQL Server 2012                │ 2022-07-12             │ ❌ HẾT 4 năm      │
│ SQL Server 2014                │ 2024-07-09             │ ❌ HẾT 2024       │
│ SQL Server 2016                │ 2026-07-14             │ ⚠️ HẾT THÁNG 7/2026│
│ SQL Server 2019                │ 2030-01-08             │ ✅ Còn ~4 năm     │
│ SQL Server 2022                │ 2033-01-11             │ ✅ Còn ~7 năm     │
└────────────────────────────────┴────────────────────────┴───────────────────┘
```

---

## 🔴 Rủi Ro Khi Tiếp Tục Dùng EOS Software

### Rủi Ro Bảo Mật

```
RỦI RO BẢO MẬT THỰC TẾ:

Case Study: WannaCry Ransomware (2017):
├── Exploit EternalBlue — lỗ hổng trong Windows SMBv1
├── Microsoft đã patch: MS17-010 (tháng 3/2017)
├── Tổ chức dùng Windows XP (đã EOS) → KHÔNG nhận patch → BỊ TẤN CÔNG
├── Thiệt hại: NHS (National Health Service Anh) tê liệt 3 ngày
│             300,000 máy tính bị mã hóa toàn cầu
└── Bài học: EOS software = không có bảo vệ khi lỗ hổng mới xuất hiện

Vòng Lặp Nguy Hiểm:
  Lỗ hổng mới phát hiện
         │
         ▼
  Microsoft KHÔNG phát hành patch cho EOS
         │
         ▼
  Hacker khai thác lỗ hổng (exploit)
         │
         ▼
  Ransomware, data breach, downtime
```

### Rủi Ro Compliance

```
COMPLIANCE VI PHẠM:

PCI-DSS (Payment Card Industry Data Security Standard —
Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán):
└── Requirement 6.3.3: Tất cả phần mềm hệ thống phải được bảo vệ
    bởi security patches. EOS software → KHÔNG PASS audit (kiểm toán)

HIPAA (Health Insurance Portability and Accountability Act —
Luật Về Tính Di Động Và Trách Nhiệm Bảo Hiểm Y Tế):
└── §164.312(a)(2)(ii): Phải có procedures để loại bỏ hoặc
    protect software không còn supported

ISO 27001 (Tiêu Chuẩn Quản Lý An Toàn Thông Tin):
└── A.12.6.1: Technical vulnerabilities phải được quản lý kịp thời.
    EOS software = unmanaged vulnerability

SOC 2 Type II:
└── Auditor sẽ flag EOS software là "exception" trong báo cáo
    → Khách hàng enterprise có thể từ chối ký hợp đồng

Hậu Quả Vi Phạm:
├── PCI-DSS: Fine (phạt) $5,000–$100,000/tháng
├── HIPAA: $100–$50,000/vi phạm, tối đa $1.9M/năm
├── GDPR: Phạt tối đa 4% doanh thu toàn cầu
└── Cyber insurance: Từ chối bồi thường do "rủi ro đã biết trước"
```

---

## 🛡️ Giải Pháp AWS: ESU Miễn Phí Trên EC2

### ESU Là Gì Và Cách Hoạt Động

```
EXTENDED SECURITY UPDATES (ESU) TRÊN AWS:

Thỏa thuận Microsoft-AWS:
├── AWS có thỏa thuận đặc biệt với Microsoft
├── Khi chạy Windows Server 2008/R2, 2012/R2 hoặc SQL Server
│   tương ứng trên EC2 → tự động nhận ESU miễn phí
├── ESU được phân phối qua Windows Update thông thường
└── Không cần cấu hình thêm — tự động

Thời Gian Hưởng ESU Miễn Phí Trên EC2:
┌──────────────────────────────────────────────┬──────────────────────────────┐
│ Phần Mềm                                     │ ESU Miễn Phí Đến            │
├──────────────────────────────────────────────┼──────────────────────────────┤
│ Windows Server 2008/R2                        │ 2023-01-10 (đã hết)         │
│ Windows Server 2012/R2                        │ 2026-10-13 (còn ~4 tháng)   │
│ SQL Server 2008/R2                            │ 2023-07-12 (đã hết)         │
│ SQL Server 2012                               │ 2025-07-12 (đã hết)         │
│ SQL Server 2014                               │ 2027-07-09 (còn ~1 năm)     │
└──────────────────────────────────────────────┴──────────────────────────────┘

Lưu ý: Nếu ESU đã hết trên EC2, phải upgrade phiên bản hoặc dùng EMP
```

### Chi Phí So Sánh: On-Premises vs EC2

```
SO SÁNH CHI PHÍ ESU:

On-Premises (phải mua ESU từ Microsoft):
├── Windows Server 2012 Standard: ~$0.0234/vCore/giờ (2023-2026)
├── Windows Server 2012 Datacenter: ~$0.1060/vCore/giờ
├── SQL Server 2014 Enterprise: ~$0.2636/vCore/giờ
└── Cho 100 servers, 8 vCore mỗi server → chi phí ESU ~$200K/năm

EC2 (ESU MIỄN PHÍ):
└── Tiết kiệm toàn bộ chi phí ESU Microsoft
    Chỉ trả EC2 compute pricing bình thường

Ví Dụ Tiết Kiệm:
├── 50 servers Windows Server 2012 Standard, 8 vCore
├── On-premises ESU: 50 × 8 × $0.0234 × 8760h = ~$82,000/năm
└── AWS: $0 ESU cost → tiết kiệm $82,000/năm chỉ từ ESU
```

### Điều Kiện Và Giới Hạn ESU Trên AWS

```
ĐIỀU KIỆN ĐỂ NHẬN ESU:

✅ Được hưởng ESU miễn phí:
├── Chạy AMI (Amazon Machine Image — Ảnh Máy Chủ Amazon) Windows chính thức từ AWS Marketplace
├── Chạy Windows Server trên EC2 instances (On-Demand, Reserved, Spot)
├── Chạy trên AWS Dedicated Hosts (Máy Chủ Chuyên Dụng)
└── BYOL (Bring Your Own License — Mang Theo License Riêng) cũng được miễn phí ESU

❌ KHÔNG được hưởng ESU miễn phí:
├── Chạy Windows trên máy on-premises
├── Chạy Windows trên Azure, GCP, hay cloud khác
├── Chạy Windows trên VMware Cloud on AWS (cần check điều khoản riêng)
└── Dùng AWS Outposts (cần xác nhận)

GIỚI HẠN:
└── ESU chỉ bao gồm: Critical và Important security patches
    KHÔNG bao gồm: Non-security bug fixes, new features
```

---

## 🔄 End-of-Support Migration Program (EMP)

### EMP Là Gì?

```
END-OF-SUPPORT MIGRATION PROGRAM (EMP):

EMP là CHƯƠNG TRÌNH, không phải single tool:
├── AWS và AWS Partners (đối tác AWS) cung cấp dịch vụ
├── Giúp nâng cấp ứng dụng từ OS/SQL cũ lên phiên bản mới
└── Tập trung vào ứng dụng KHÔNG tương thích trực tiếp với OS mới

EMP Bao Gồm:
├── Assessment: Đánh giá tương thích ứng dụng với OS mới
├── Remediation: Sửa code/configuration để tương thích
├── Packaging: Đóng gói ứng dụng với virtualization layer
│   dùng AWS End-of-Support Migration Agent (công nghệ từ Orca)
├── Testing: Kiểm thử trên OS mới
└── Deployment: Deploy lên EC2 với OS mới
```

### EMP Agent — Công Nghệ Cốt Lõi

```
EMP AGENT — VIRTUALIZATION SHIM:

Vấn đề: App Windows XP/2003 phụ thuộc vào Windows Registry keys
         và DLL cũ → không chạy được trên Windows Server 2019/2022

Giải pháp EMP Agent:
┌────────────────────────────────────────────────────────────────────┐
│ Windows Server 2019/2022 (OS MỚI)                                  │
│                                                                    │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │ EMP Agent — Virtualization Layer (Lớp Ảo Hóa)          │      │
│   │ ├── Intercept API calls từ ứng dụng                     │      │
│   │ ├── Redirect deprecated Registry keys → new equivalents │      │
│   │ ├── Provide compatibility layer for old DLLs            │      │
│   │ └── Isolate app trong virtual environment               │      │
│   │                                                         │      │
│   │   ┌─────────────────────────────────────────────────┐   │      │
│   │   │ Legacy Application (viết cho Windows XP/2003)   │   │      │
│   │   │ Chạy "tưởng" mình đang trên Windows cũ          │   │      │
│   │   └─────────────────────────────────────────────────┘   │      │
│   └─────────────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────────┘

Kết quả: App chạy trên OS mới mà KHÔNG cần sửa code
```

### Khi Nào Dùng EMP vs Upgrade Trực Tiếp

```
QUYẾT ĐỊNH:

Upgrade trực tiếp (không cần EMP) khi:
├── App hiện đại (sau 2010), vendor vẫn hỗ trợ
├── App tương thích với Windows Server 2019/2022 (test trước)
├── Source code available → có thể recompile với target mới
└── App dùng standard APIs, không dùng deprecated features

Dùng EMP khi:
├── App phụ thuộc vào deprecated Windows components
│   (ví dụ: DCOM, 16-bit components, old COM DLLs)
├── Không có source code (vendor đã phá sản hoặc mất code)
├── App COTS (Commercial Off-The-Shelf — Phần Mềm Thương Mại Sẵn Có)
│   mà vendor không có bản mới hơn
├── App tương thích từng được kiểm tra nhưng có breaking changes
└── Muốn migrate nhanh, không muốn đầu tư refactoring
```

---

## 🪟 Lộ Trình Upgrade Windows Server

### Upgrade In-Place vs Fresh Install

```
HAI PHƯƠNG PHÁP UPGRADE:

┌──────────────────────────────┬──────────────────────────────────────────────┐
│ In-Place Upgrade             │ Side-by-Side Migration                      │
│ (Nâng Cấp Tại Chỗ)          │ (Di Chuyển Song Song)                        │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Nâng cấp OS trực tiếp trên  │ Cài OS mới trên server/EC2 mới,              │
│ instance hiện tại            │ migrate app sang đó                          │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Ưu: Đơn giản, ít bước        │ Ưu: Không rủi ro cho server gốc,            │
│                              │     có thể test trước khi cutover            │
│ Nhược: Nếu fail → downtime   │ Nhược: Tốn thêm resources trong thời gian   │
│        khó rollback           │        migration                            │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Phù hợp: Development/staging │ Phù hợp: Production workloads quan trọng    │
│          environments        │          (khuyến nghị)                      │
└──────────────────────────────┴──────────────────────────────────────────────┘
```

### Con Đường Upgrade Hợp Lệ (Supported Upgrade Paths)

```
WINDOWS SERVER UPGRADE PATHS:

Windows Server 2012 R2 → 2016 → 2019 → 2022
(Không thể nhảy cóc qua nhiều phiên bản: 2012 → 2022 TRỰC TIẾP KHÔNG ĐƯỢC)

Ví dụ:
├── WS 2012 R2 → WS 2016 (OK)
├── WS 2012 R2 → WS 2019 (OK trên cùng kiến trúc)
├── WS 2012 R2 → WS 2022 (KHÔNG OK trực tiếp — phải qua 2016 hoặc 2019)
├── WS 2016 → WS 2019 (OK)
├── WS 2016 → WS 2022 (OK)
└── WS 2019 → WS 2022 (OK)

TRÊN AWS — Phương pháp khuyến nghị (Side-by-Side):
1. Tạo EC2 instance mới với Windows Server 2022 AMI
2. Cài và configure ứng dụng trên server mới
3. Test kỹ lưỡng
4. Cutover DNS/Load balancer
5. Terminate (chấm dứt) EC2 cũ sau khi ổn định
```

### Thực Hiện Upgrade Trên AWS

```
QUY TRÌNH UPGRADE WINDOWS SERVER TRÊN AWS (SIDE-BY-SIDE):

Bước 1: Chuẩn Bị
  □ Snapshot AMI (tạo bản sao ảnh máy) của EC2 hiện tại
  □ Document tất cả installed apps, services, configurations
  □ Export IIS config: appcmd.exe list site /xml > sites.xml
  □ Export scheduled tasks: schtasks /query /fo XML > tasks.xml
  □ Document firewall rules

Bước 2: Tạo Server Mới
  □ Launch EC2 từ Windows Server 2022 AMI mới nhất
  □ Chọn instance type phù hợp (có thể right-size)
  □ Configure Security Groups giống server cũ
  □ Kết nối vào domain (nếu dùng Active Directory)

Bước 3: Cài Đặt Ứng Dụng
  □ Cài .NET Framework / .NET runtime cần thiết
  □ Cài IIS với modules tương tự server cũ
  □ Import IIS config từ backup
  □ Deploy application code
  □ Configure environment variables và connection strings
  □ Cài các dependencies (ứng dụng bên thứ ba)

Bước 4: Testing
  □ Smoke test: Tất cả services khởi động đúng
  □ Functional test: Tất cả features hoạt động đúng
  □ Performance test: Load testing so sánh với server cũ
  □ Security scan: Windows Defender, patch level check

Bước 5: Cutover
  □ Maintenance window (cửa sổ bảo trì) ngoài giờ cao điểm
  □ Final data/config sync từ server cũ
  □ Update Route 53 / Load Balancer target
  □ Monitor 30 phút sau cutover
  □ Nếu lỗi: Rollback về server cũ ngay

Bước 6: Decommission (Ngừng Vận Hành)
  □ Monitor server mới 1-2 tuần
  □ Stop (dừng) EC2 cũ (chưa terminate)
  □ Sau 30 ngày không có vấn đề: Terminate EC2 cũ
```

---

## 🗄️ Lộ Trình Upgrade SQL Server

### Upgrade SQL Server Tại Chỗ vs Migrate Sang RDS

```
HAI HƯỚNG ĐI CHO SQL SERVER EOS:

┌──────────────────────────────┬──────────────────────────────────────────────┐
│ Hướng 1: Upgrade SQL Server  │ Hướng 2: Migrate sang Amazon RDS            │
│ lên phiên bản mới trên EC2   │ hoặc Amazon Aurora                          │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ SQL Server 2014 → 2022       │ SQL Server → RDS SQL Server 2022            │
│ Vẫn trên EC2 Windows         │ hoặc SQL Server → Aurora PostgreSQL         │
│                              │ (với AWS DMS + SCT)                          │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Ưu: Không thay đổi app code  │ Ưu: Managed service — không quản lý OS,    │
│                              │     tự động backups, Multi-AZ HA,           │
│                              │     tiết kiệm DBA effort                    │
│ Nhược: Vẫn manage OS,        │ Nhược: Có thể cần sửa app connection string;│
│        patching, backups tự  │        Aurora migration cần nhiều effort     │
│        tay                   │        hơn (stored procs, syntax khác)      │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Phù hợp: App phức tạp với    │ Phù hợp: Muốn giảm operational burden;     │
│ nhiều SQL Server features     │ app chuẩn, ít stored procedures            │
│ đặc thù                      │                                             │
└──────────────────────────────┴──────────────────────────────────────────────┘
```

### Upgrade SQL Server In-Place Trên EC2

```
SQL SERVER IN-PLACE UPGRADE TRÊN EC2:

Supported Upgrade Paths:
├── SQL Server 2014 → 2019 (OK, nhảy được)
├── SQL Server 2014 → 2022 (OK)
├── SQL Server 2016 → 2019 (OK)
├── SQL Server 2016 → 2022 (OK)
└── SQL Server 2019 → 2022 (OK)

Quy Trình:
1. Backup toàn bộ databases (Full backup + Transaction log backup)
2. Create AMI snapshot của EC2 instance (rollback plan)
3. Download SQL Server 2022 installation media
4. Chạy SQL Server Setup với option "Upgrade"
5. Validate: Chạy DBCC CHECKDB trên tất cả databases
6. Test ứng dụng
7. Update maintenance plans và backup jobs

Lưu Ý:
├── In-place upgrade: Downtime khoảng 30-60 phút
├── SQL Server Agent jobs, linked servers, certificates cần verify sau upgrade
└── Compatibility level: Có thể giữ old compat level ban đầu để tránh breaking changes
    ALTER DATABASE mydb SET COMPATIBILITY_LEVEL = 150  -- SQL 2019
    (Nâng dần sau khi test kỹ)
```

### Migrate SQL Server → Amazon RDS Với DMS

```
SQL SERVER ON EC2 → AMAZON RDS SQL SERVER:

Dùng AWS DMS (Database Migration Service):
├── Source: SQL Server 2014 trên EC2
├── Target: RDS SQL Server 2022
├── Phương pháp: Full load + CDC (Change Data Capture)
└── Downtime: Tối thiểu (chỉ final cutover)

Lưu Ý Về Compatibility:
├── RDS SQL Server KHÔNG hỗ trợ:
│   ├── SQL Server Agent (thay bằng Amazon EventBridge + Lambda)
│   ├── Database Mail (thay bằng Amazon SES)
│   ├── Windows Authentication to domain users
│   │   (chỉ hỗ trợ SQL authentication hoặc IAM authentication)
│   └── BULK INSERT từ local file system (dùng S3 thay)
│
└── RDS SQL Server HỖ TRỢ:
    ├── Stored procedures, functions, views
    ├── CLR (Common Language Runtime) assemblies (một số)
    ├── Linked servers (với limitations)
    └── Transparent Data Encryption (TDE — Mã Hóa Dữ Liệu Trong Suốt)

Sau khi migrate thành công:
├── Không còn quản lý Windows OS
├── Automatic backups lên S3 theo retention policy
├── Multi-AZ: Tự động failover (chuyển đổi dự phòng) nếu primary fail
└── Performance Insights để monitor query performance
```

---

## 📊 So Sánh Các Phương Án

```
MA TRẬN QUYẾT ĐỊNH CHO EOS WORKLOADS:

┌─────────────────┬──────────────────┬───────────────────────┬──────────────────────┐
│ Tiêu Chí        │ ESU Trên EC2     │ In-Place Upgrade      │ Migrate → RDS/Aurora │
│                 │ (Tạm Thời)       │ Trên EC2              │ (Managed Service)    │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Thời gian       │ 1-2 tuần         │ 2-4 tuần              │ 4-12 tuần            │
│ thực hiện       │                  │                       │                      │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Rủi ro          │ Thấp             │ Trung bình            │ Trung bình đến cao   │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Thay đổi app    │ Không            │ Có thể cần            │ Có thể cần sửa       │
│ code            │                  │ testing               │ connection strings   │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Lợi ích dài hạn │ Không — vẫn phải │ Trung bình — vẫn      │ Cao — không quản lý  │
│                 │ upgrade cuối     │ manage OS/SQL         │ OS, auto patching    │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Chi phí         │ Thấp (chỉ EC2)   │ Trung bình            │ Có thể cao hơn EC2   │
│ vận hành        │                  │ (EC2 + license)       │ tùy workload         │
├─────────────────┼──────────────────┼───────────────────────┼──────────────────────┤
│ Khi nào         │ Cần thêm thời    │ App phức tạp,         │ Muốn minimize        │
│ dùng            │ gian để plan,    │ không muốn managed    │ operational work,    │
│                 │ ESU còn hiệu lực │ service               │ app tiêu chuẩn       │
└─────────────────┴──────────────────┴───────────────────────┴──────────────────────┘
```

---

## 📋 Compliance Và Audit

### Chứng Minh Compliance Sau EOS Migration

```
BẰNG CHỨNG CHO AUDITOR (KIỂM TOÁN VIÊN):

Sau khi migrate lên OS/SQL mới hoặc lên managed service:

1. AWS Config Rules:
   └── Bật rule: restricted-common-ports
       ec2-instances-in-vpc
       Tạo custom rule kiểm tra Windows AMI version

2. AWS Systems Manager Patch Manager:
   └── Báo cáo patch compliance: tất cả EC2 instances
       phải ở trạng thái "Compliant" (tuân thủ)
       Export báo cáo cho auditor

3. Amazon Inspector:
   └── Vulnerability assessment scan định kỳ
       Báo cáo CVE (Common Vulnerabilities and Exposures —
       Lỗ Hổng Và Rủi Ro Phổ Biến) và mức độ nghiêm trọng

4. AWS Security Hub:
   └── Dashboard compliance tổng hợp
       Tích hợp với CIS Benchmarks (Center for Internet Security)
       Tích hợp với PCI-DSS controls

5. CloudTrail:
   └── Audit log đầy đủ: Ai làm gì, khi nào
       Bằng chứng rằng patching được thực hiện đúng quy trình
```

### Patch Management Tự Động

```
AWS SYSTEMS MANAGER PATCH MANAGER — TỰ ĐỘNG VÁ LỖI:

Cấu Hình:
1. Tạo Patch Baseline (Đường Cơ Sở Vá Lỗi):
   - Chọn: Critical và Important patches được auto-approve sau 7 ngày
   - Exclude: Specific patches nếu cần (ví dụ: breaking change)

2. Tạo Maintenance Window (Cửa Sổ Bảo Trì):
   - Thứ Ba, 2:00 AM - 4:00 AM (ít traffic nhất)
   - Duration: 2 giờ

3. Assign vào EC2 Instances:
   - Dùng Tags để chọn instances: Environment=Production
   - Run Command: AWS-RunPatchBaseline

4. Báo Cáo:
   - Patch compliance dashboard
   - Non-compliant instances được alert qua SNS
   - Export CSV để nộp cho auditor

Kết Quả:
└── Không bao giờ chạy EOS software → không còn compliance violation
    Có bằng chứng audit trail đầy đủ
```

---

## ✅ Checklist EOS Migration

### Giai Đoạn Phát Hiện (Discovery)

```
□ Inventory toàn bộ Windows Server và SQL Server versions:
  Dùng AWS Systems Manager Inventory hoặc ADS
□ Lọc ra instances đang chạy phiên bản EOS hoặc sắp EOS trong 12 tháng
□ Phân loại theo mức độ nghiêm trọng:
  - Đã hết hỗ trợ → Ưu tiên cao nhất (immediate action)
  - Hết trong 6 tháng → Ưu tiên cao
  - Hết trong 12 tháng → Lên kế hoạch ngay
□ Kiểm tra ESU còn hiệu lực không (xem bảng timeline)
□ Xác định dependency: App nào phụ thuộc vào từng server/database
```

### Giai Đoạn Lập Kế Hoạch (Planning)

```
□ Quyết định phương án cho từng workload: ESU / Upgrade / RDS / EMP
□ Đánh giá tương thích ứng dụng với OS/SQL mới (compatibility testing)
□ Ước tính timeline và resource cần thiết
□ Lên lịch maintenance windows
□ Chuẩn bị rollback plan cho từng migration
□ Thông báo stakeholders về planned downtime (thời gian ngừng dịch vụ theo kế hoạch)
□ Xây dựng test plan cho ứng dụng trên phiên bản mới
```

### Giai Đoạn Thực Hiện (Execution)

```
□ Tạo AMI snapshot trước mọi thay đổi
□ Thực hiện migration theo kế hoạch
□ Chạy full test suite trên phiên bản mới
□ Validate performance: Không thấp hơn phiên bản cũ
□ Xác nhận security patches đang nhận đúng
□ Cập nhật CMDB (Configuration Management Database — Cơ Sở Dữ Liệu Quản Lý Cấu Hình)
□ Cập nhật documentation (tài liệu) và runbooks
□ Decommission server/instance cũ sau 30 ngày ổn định
```

### Giai Đoạn Validate Compliance

```
□ Chạy AWS Config compliance check
□ Chạy Amazon Inspector vulnerability scan — zero Critical CVE
□ Export Systems Manager Patch Manager compliance report
□ Review Security Hub findings — không có EOS-related violations
□ Cập nhật security policy documentation
□ Thông báo cho CISO và compliance team: Migration hoàn thành
□ Archive evidence cho audit trail
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Extended Security Updates (ESU) miễn phí trên AWS EC2 hoạt động như thế nào?**

> Khi chạy Windows Server 2008/R2 hoặc 2012/R2 (và các phiên bản SQL Server tương ứng) trên Amazon EC2, AWS tự động cung cấp Extended Security Updates miễn phí thông qua Windows Update thông thường — không cần cấu hình thêm.
>
> Điều này dựa trên thỏa thuận đặc biệt giữa AWS và Microsoft. Trên on-premises, doanh nghiệp phải mua ESU từ Microsoft với giá từ vài cent đến vài chục cent mỗi vCore mỗi giờ.
>
> Thực tế quan trọng: ESU chỉ là giải pháp **tạm thời** — chỉ cung cấp security patches, không có feature mới. Tổ chức vẫn cần lên kế hoạch upgrade lên phiên bản được hỗ trợ đầy đủ trong tương lai.

---

**Q: Sự khác biệt giữa in-place upgrade và side-by-side migration cho Windows Server EOS?**

> **In-place upgrade** (nâng cấp tại chỗ): Nâng cấp OS trực tiếp trên instance hiện tại. Đơn giản hơn nhưng nếu upgrade thất bại giữa chừng → downtime kéo dài, rollback phức tạp.
>
> **Side-by-side migration** (di chuyển song song): Tạo EC2 instance mới với OS mới, cài app lên đó, test kỹ, rồi mới chuyển traffic. An toàn hơn nhiều: Server cũ vẫn chạy song song; nếu có vấn đề chỉ cần chuyển traffic về server cũ ngay lập tức.
>
> Khuyến nghị: Luôn dùng side-by-side cho production workloads. Chỉ dùng in-place khi app quá phức tạp để rebuild, hoặc trên non-production environments.

---

**Q: Tại sao không thể upgrade trực tiếp từ Windows Server 2012 lên Windows Server 2022?**

> Microsoft chỉ hỗ trợ upgrade theo "supported upgrade paths" — thường chỉ nhảy được 1-2 phiên bản. Lý do kỹ thuật: quá nhiều thay đổi giữa các phiên bản cách xa — registry, system services, deprecated components — khiến upgrade wizard không thể handle an toàn.
>
> Thực tế: Với AWS EC2, vấn đề này ít quan trọng hơn vì ta thường dùng side-by-side migration (tạo EC2 mới với OS mới) thay vì in-place upgrade. Không bị giới hạn bởi upgrade paths — có thể tạo Windows Server 2022 instance mới và migrate app từ 2012 lên 2022 trực tiếp.

---

### Câu Hỏi Nâng Cao

**Q: Công ty có 200 Windows Server 2012 R2 instances đang chạy EOS. Bạn sẽ tiếp cận vấn đề này như thế nào?**

> Tôi sẽ tiếp cận theo 4 giai đoạn:
>
> **1. Discover & Categorize (Tuần 1-2)**:
> - Dùng AWS Systems Manager Inventory để lấy danh sách đầy đủ
> - Phân loại theo criticality: production vs non-production, workload type
> - Kiểm tra ESU còn hiệu lực không (Windows Server 2012 ESU đến tháng 10/2026)
>
> **2. Lập chiến lược từng nhóm (Tuần 2-3)**:
> - Non-production → upgrade ngay, ít rủi ro
> - Production đơn giản → side-by-side upgrade lên WS 2022
> - Production phức tạp với app legacy → đánh giá EMP
> - Servers sắp retire trong 12 tháng → chỉ cần ESU tạm, không migrate
>
> **3. Thực hiện theo wave (Tuần 4+)**:
> - Wave 1: Non-production (thử nghiệm và học)
> - Wave 2: Production ít critical
> - Wave 3: Core production systems
>
> **4. Automation**:
> - Dùng AWS Systems Manager Automation để chuẩn hóa quy trình
> - Infrastructure as Code (CloudFormation/Terraform) cho environment mới
> - Patch Manager để duy trì compliance sau này
>
> Mục tiêu: Hoàn thành 100% trước tháng 10/2026 (khi ESU hết).

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 08-modernization
