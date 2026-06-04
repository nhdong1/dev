# AWS Snow Family — Di Chuyển Dữ Liệu Ngoại Tuyến: Tổng Quan & So Sánh

> **AWS Snow Family** là bộ thiết bị vật lý (physical devices) do AWS cung cấp để di chuyển dữ liệu khối lượng lớn theo hình thức **ngoại tuyến** (offline) — nghĩa là dữ liệu được chép vào thiết bị tại chỗ, vận chuyển về trung tâm dữ liệu AWS, rồi import vào S3. Snow Family giải quyết vấn đề băng thông mạng hạn chế, latency cao, hoặc chi phí truyền tải quá lớn khi dùng internet.

## 📚 Mục Lục (Table of Contents)

1. [Vấn Đề Snow Family Giải Quyết](#vấn-đề-snow-family-giải-quyết)
2. [Ba Thành Viên Snow Family](#ba-thành-viên-snow-family)
3. [Bảng So Sánh Tổng Hợp](#bảng-so-sánh-tổng-hợp)
4. [Quy Trình Hoạt Động Chung](#quy-trình-hoạt-động-chung)
5. [Bảo Mật Snow Family](#bảo-mật-snow-family)
6. [Cách Chọn Đúng Thiết Bị](#cách-chọn-đúng-thiết-bị)
7. [AWS OpsHub — Giao Diện Quản Lý](#aws-opshub--giao-diện-quản-lý)
8. [Tích Hợp Với Các Dịch Vụ AWS](#tích-hợp-với-các-dịch-vụ-aws)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Vấn Đề Snow Family Giải Quyết

### Tại Sao Không Dùng Internet Để Chuyển Dữ Liệu?

```
Bài toán thực tế:
Bạn có 100 TB dữ liệu cần chuyển lên AWS.
Kết nối internet của bạn: 1 Gbps (125 MB/s)

Tính toán thời gian:
├── 100 TB = 100 × 1024 GB = 102,400 GB
├── Tốc độ thực tế: 1 Gbps ÷ 8 = 125 MB/s
├── Thời gian lý thuyết: 102,400 GB ÷ 0.125 GB/s = 819,200 giây
└── ≈ 9.5 ngày (lý thuyết, thực tế còn lâu hơn)

Vấn đề thực tế khi dùng internet:
├── Băng thông dùng chung với traffic khác (production bị ảnh hưởng)
├── Chi phí data transfer ra internet (egress fees)
├── Kết nối không ổn định → phải retry nhiều lần
├── Vùng xa không có internet tốc độ cao
└── Security concerns khi truyền dữ liệu nhạy cảm qua internet

Quy tắc của AWS:
"Nếu tải dữ liệu lên AWS mất hơn 1 tuần qua mạng → Hãy dùng Snow Family"
```

### Ngưỡng Quyết Định: Network vs Snow Family

```
┌─────────────────────────────────────────────────────────────────────┐
│              NGƯỠNG QUYẾT ĐỊNH (Decision Threshold)                 │
├──────────────┬──────────────────┬──────────────────────────────────┤
│ Băng Thông   │ Thời Gian 10 TB  │ Khuyến Nghị                      │
├──────────────┼──────────────────┼──────────────────────────────────┤
│ 10 Mbps      │ ~111 ngày        │ ✅ Dùng Snow Family               │
│ 100 Mbps     │ ~11 ngày         │ ✅ Dùng Snow Family               │
│ 1 Gbps       │ ~1 ngày          │ ⚠️  Tùy thuộc (Snow nhanh hơn)   │
│ 10 Gbps      │ ~2.4 giờ         │ ✅ Dùng Network (DataSync)        │
└──────────────┴──────────────────┴──────────────────────────────────┘

Nguyên tắc: Nếu upload mất > 7 ngày → dùng Snow Family
```

---

## ❄️ Ba Thành Viên Snow Family

### 1. AWS Snowcone — Thiết Bị Nhỏ Gọn Nhất

```
┌────────────────────────────────────────────────────────────┐
│                    AWS SNOWCONE                             │
│                                                             │
│  Kích thước: Nhỏ như hộp cơm (227 × 148 × 82 mm)          │
│  Trọng lượng: 2.1 kg                                        │
│  Dung lượng:                                                │
│  ├── HDD: 8 TB (usable — sử dụng được)                     │
│  └── SSD: 14 TB (usable)                                    │
│                                                             │
│  Tính năng đặc biệt:                                        │
│  ├── Chạy được bằng pin hoặc nguồn USB-C                    │
│  ├── Chịu được điều kiện môi trường khắc nghiệt             │
│  ├── Có thể dùng DataSync để transfer qua mạng              │
│  └── Có thể chạy EC2 instances (edge computing)             │
└────────────────────────────────────────────────────────────┘
```

**Dùng khi:** Vùng xa, không gian hạn chế, IoT/edge, thu thập dữ liệu thực địa

**Chi tiết:** [1-snowcone.md](./1-snowcone.md)

---

### 2. AWS Snowball Edge — Dòng Chủ Lực

```
┌────────────────────────────────────────────────────────────┐
│                  AWS SNOWBALL EDGE                          │
│                                                             │
│  Hai phiên bản:                                             │
│                                                             │
│  Storage Optimized (Tối Ưu Lưu Trữ):                       │
│  ├── Storage: 80 TB HDD (usable)                            │
│  ├── RAM: 80 GB                                             │
│  └── vCPU: 40 (cho edge computing)                         │
│                                                             │
│  Compute Optimized (Tối Ưu Tính Toán):                     │
│  ├── Storage: 28 TB SSD (usable) hoặc 42 TB HDD            │
│  ├── RAM: 416 GB                                            │
│  └── vCPU: 52 + tùy chọn GPU (NVIDIA Tesla V100)           │
│                                                             │
│  Cả hai đều có:                                             │
│  ├── Chạy EC2 instances AMI                                 │
│  ├── Chạy AWS Lambda functions                              │
│  └── Hỗ trợ clustering (tập hợp) nhiều thiết bị            │
└────────────────────────────────────────────────────────────┘
```

**Dùng khi:** Di chuyển dữ liệu 10-80 TB, edge computing công nghiệp, ML inference tại chỗ

**Chi tiết:** [2-snowball-edge.md](./2-snowball-edge.md)

---

### 3. AWS Snowmobile — Xe Tải Dữ Liệu

```
┌────────────────────────────────────────────────────────────┐
│                   AWS SNOWMOBILE                            │
│                                                             │
│  Dạng: Container 45 feet (≈ 13.7 m) kéo bởi xe tải        │
│  Dung lượng: 100 PB (petabyte) mỗi xe                       │
│  Bảo vệ: GPS tracking, CCTV, nhân viên bảo vệ, alarm       │
│                                                             │
│  Điều kiện sử dụng:                                         │
│  ├── Dữ liệu > 10 PB                                        │
│  ├── Datacenter consolidation (gộp nhiều DC lại)            │
│  └── Exabyte-scale migration (di chuyển exabyte)           │
│                                                             │
│  Quy trình đặc biệt:                                        │
│  ├── AWS giao xe đến địa điểm của bạn                       │
│  ├── Kết nối vào mạng nội bộ datacenter                     │
│  ├── Chép dữ liệu trực tiếp (tốc độ đến 1 Tbps)            │
│  └── AWS lái xe về và import vào S3                         │
└────────────────────────────────────────────────────────────┘
```

**Dùng khi:** > 10 PB, consolidating multiple datacenters, exabyte migration

**Chi tiết:** [3-snowmobile.md](./3-snowmobile.md)

---

## 📊 Bảng So Sánh Tổng Hợp

### So Sánh Thông Số Kỹ Thuật

| Thuộc Tính                            | Snowcone HDD     | Snowcone SSD     | Snowball Edge Storage | Snowball Edge Compute | Snowmobile     |
| ------------------------------------- | ---------------- | ---------------- | --------------------- | --------------------- | -------------- |
| **Dung lượng (usable)**               | 8 TB             | 14 TB            | 80 TB                 | 28 TB SSD / 42 TB HDD | 100 PB         |
| **vCPU**                              | 2                | 2                | 40                    | 52 (+ GPU tùy chọn)   | N/A            |
| **RAM**                               | 4 GB             | 4 GB             | 80 GB                 | 416 GB                | N/A            |
| **GPU**                               | ✗                | ✗                | ✗                     | ✅ NVIDIA V100        | N/A            |
| **EC2 Instances**                     | ✅ (micro)       | ✅ (micro)       | ✅                    | ✅                    | ✗              |
| **Lambda Functions**                  | ✅               | ✅               | ✅                    | ✅                    | ✗              |
| **Storage Clustering**                | ✗                | ✗                | ✅ (tối đa 15 nodes)  | ✅ (tối đa 15 nodes)  | N/A            |
| **Pin/Battery**                       | ✅               | ✅               | ✗                     | ✗                     | ✗              |
| **Kích thước**                        | Hộp cơm          | Hộp cơm          | Vali nhỏ              | Vali nhỏ              | Container 45ft |
| **Thời gian dùng tối đa**             | 360 ngày/lần     | 360 ngày/lần     | 360 ngày/lần          | 360 ngày/lần          | Linh hoạt      |

### So Sánh Use Cases (Trường Hợp Sử Dụng)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CHỌN THIẾT BỊ NÀO?                           │
├──────────────────────────────┬───────────────────────────────────────┤
│ Tình Huống                   │ Thiết Bị Phù Hợp                      │
├──────────────────────────────┼───────────────────────────────────────┤
│ < 14 TB, vùng xa, di động    │ Snowcone                               │
│ 14 - 80 TB, cần lưu trữ     │ Snowball Edge Storage Optimized        │
│ < 42 TB, cần GPU/ML          │ Snowball Edge Compute Optimized        │
│ 80 TB - 10 PB (nhiều thiết  │ Nhiều Snowball Edge (Clustering)       │
│ bị cùng lúc)                 │                                        │
│ > 10 PB                      │ Snowmobile                             │
└──────────────────────────────┴───────────────────────────────────────┘
```

### So Sánh Bảo Mật

| Tính Năng Bảo Mật                                   | Snowcone | Snowball Edge | Snowmobile |
| --------------------------------------------------- | -------- | ------------- | ---------- |
| **Mã hóa AES-256** (256-bit AES Encryption)         | ✅       | ✅            | ✅         |
| **KMS** (Key Management Service — Quản Lý Khóa)    | ✅       | ✅            | ✅         |
| **TPM** (Trusted Platform Module — Module Bảo Mật) | ✅       | ✅            | ✅         |
| **Tamper-evident** (Phát hiện bị mở/giả mạo)       | ✅       | ✅            | ✅         |
| **Secure erase** sau khi import xong                | ✅       | ✅            | ✅         |
| **GPS Tracking**                                    | ✗        | ✗             | ✅         |
| **CCTV + Bảo Vệ Vũ Trang**                         | ✗        | ✗             | ✅         |

---

## 🔄 Quy Trình Hoạt Động Chung (Snow Family Workflow)

```
BƯỚC 1: ĐẶT HÀNG (ORDER)
┌─────────────────────────────────────────────────────────────────┐
│  Truy cập AWS Console → Snow Family → Create Job                 │
│  Chọn: Import into S3 / Export from S3 / Local compute & storage│
│  Chỉ định: S3 bucket đích, dung lượng cần, địa chỉ giao hàng   │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 2: AWS CHUẨN BỊ VÀ GIAO THIẾT BỊ
┌─────────────────────────────────────────────────────────────────┐
│  AWS chuẩn bị thiết bị và giao đến địa điểm của bạn             │
│  (thường 2-7 ngày làm việc tùy khu vực)                          │
│  Kèm theo: manifest file (file kê khai) và unlock code          │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 3: COPY DỮ LIỆU VÀO THIẾT BỊ
┌─────────────────────────────────────────────────────────────────┐
│  Kết nối thiết bị vào mạng nội bộ (LAN)                         │
│  Unlock thiết bị bằng AWS OpsHub hoặc Snowball client            │
│  Copy dữ liệu bằng:                                             │
│  ├── AWS Snowball client (CLI tool)                              │
│  ├── S3-compatible API                                           │
│  ├── NFS mount (Snowball Edge)                                   │
│  └── AWS OpsHub GUI                                             │
│  Tốc độ copy: 1-10 Gbps tùy kết nối và thiết bị                │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 4: GỬI TRẢ THIẾT BỊ VỀ AWS
┌─────────────────────────────────────────────────────────────────┐
│  Đóng gói thiết bị (có E-ink label tự cập nhật địa chỉ)        │
│  Giao cho đơn vị vận chuyển (UPS, DHL, v.v.)                   │
│  Theo dõi trạng thái qua AWS Console                            │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 5: AWS IMPORT DỮ LIỆU VÀO S3
┌─────────────────────────────────────────────────────────────────┐
│  AWS nhận thiết bị tại AWS facility (cơ sở)                     │
│  Giải mã và import dữ liệu vào S3 bucket của bạn               │
│  Sau khi hoàn tất: thông báo qua email/SNS                      │
│  AWS thực hiện secure erase (xóa an toàn) thiết bị              │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
BƯỚC 6: XÁC NHẬN DỮ LIỆU
┌─────────────────────────────────────────────────────────────────┐
│  Kiểm tra S3 bucket — tất cả object đã ở đúng vị trí?           │
│  Xác minh checksum (kiểm tra tổng băm) nếu có yêu cầu          │
│  Cập nhật ứng dụng để trỏ vào S3 thay vì on-premises           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Bảo Mật Snow Family

### Kiến Trúc Bảo Mật Nhiều Lớp

```
LỚP 1: MÃ HÓA DỮ LIỆU (Data Encryption)
├── Tất cả dữ liệu được mã hóa AES-256 trước khi ghi vào thiết bị
├── Khóa mã hóa (encryption key) được quản lý bởi AWS KMS
├── Bạn sở hữu customer master key (CMK — Khóa Chính Khách Hàng)
└── AWS không bao giờ có quyền truy cập vào CMK của bạn

LỚP 2: BẢO VỆ VẬT LÝ (Physical Security)
├── TPM — Trusted Platform Module (Module Bảo Mật Tin Cậy)
│   └── Chip phần cứng xác minh thiết bị chưa bị giả mạo
├── Tamper-evident enclosure (vỏ bọc phát hiện can thiệp)
│   └── Nếu thiết bị bị mở, TPM sẽ khóa ngay lập tức
├── E-ink display (màn hình mực điện tử) cho shipping label
└── HIPAA, PCI-DSS, SOC compliant (đạt tiêu chuẩn)

LỚP 3: KIỂM SOÁT TRUY CẬP (Access Control)
├── Unlock code (mã mở khóa) chỉ bạn nhận được qua email
├── Manifest file (file kê khai) xác thực thiết bị
├── IAM roles (vai trò IAM) kiểm soát quyền copy data
└── Job logging qua AWS CloudTrail

LỚP 4: XÓA AN TOÀN (Secure Erase)
├── Sau khi import hoàn tất, AWS thực hiện NIST 800-88
│   (National Institute of Standards and Technology — Viện Tiêu Chuẩn Quốc Gia)
└── Chứng nhận xóa được cung cấp nếu cần
```

---

## 🎯 Cách Chọn Đúng Thiết Bị

### Decision Tree (Cây Quyết Định)

```
Bạn cần di chuyển bao nhiêu dữ liệu?
│
├── < 14 TB?
│   ├── Cần edge computing hoặc vùng rất xa? → Snowcone
│   └── Có băng thông tốt? → DataSync (nhanh hơn)
│
├── 14 TB – 80 TB?
│   ├── Ưu tiên lưu trữ nhiều nhất → Snowball Edge Storage Optimized
│   └── Cần ML/AI inference tại chỗ → Snowball Edge Compute Optimized
│
├── 80 TB – 10 PB?
│   └── Nhiều Snowball Edge, cluster lại (tối đa 15 nodes = 1.2 PB)
│
└── > 10 PB?
    └── Snowmobile (100 PB/xe, có thể dùng nhiều xe)
```

### Bảng Quyết Định Nhanh

| Kịch Bản                                              | Giải Pháp Được Khuyến Nghị         |
| ----------------------------------------------------- | ---------------------------------- |
| Thu thập sensor data ngoài thực địa (field)           | Snowcone                           |
| IoT data collection vùng không có internet            | Snowcone                           |
| Di chuyển 50 TB on-premises NAS lên S3                | Snowball Edge Storage Optimized    |
| ML model training tại nhà máy sản xuất               | Snowball Edge Compute Optimized    |
| Backup 200 TB sang AWS Glacier                        | 3x Snowball Edge Storage Optimized |
| Consolidation toàn bộ datacenter (10+ PB)             | Snowmobile                         |
| Exabyte-scale media archive migration                 | Snowmobile (nhiều xe)              |

---

## 🖥️ AWS OpsHub — Giao Diện Quản Lý

```
AWS OpsHub = GUI application (ứng dụng giao diện đồ họa)
để quản lý Snow Family devices — thay thế cho Snowball CLI

Cài đặt: Windows hoặc macOS

Tính năng:
├── Unlock thiết bị
├── Xem status (trạng thái) thiết bị
├── Transfer files bằng drag-and-drop (kéo thả)
├── Quản lý EC2 instances chạy trên thiết bị
├── Monitor (theo dõi) hiệu suất và dung lượng
└── Cấu hình network interfaces (giao diện mạng)

Trước OpsHub: Phải dùng Snowball client CLI
Sau OpsHub:  GUI thân thiện, phù hợp cả người không rành CLI
```

---

## 🔗 Tích Hợp Với Các Dịch Vụ AWS

```
Snow Family ←→ Dịch Vụ AWS

S3 (Simple Storage Service — Dịch Vụ Lưu Trữ Đơn Giản):
└── Điểm đến chính sau khi import — dữ liệu vào S3 bucket

S3 Glacier (Lưu trữ lạnh, chi phí thấp):
└── Sau khi vào S3, có thể lifecycle policy chuyển sang Glacier

EC2 (Elastic Compute Cloud — Điện Toán Đám Mây Linh Hoạt):
└── Snowball Edge và Snowcone có thể chạy EC2 AMIs tại chỗ

Lambda (Serverless Functions — Hàm Không Máy Chủ):
└── Chạy Lambda functions trực tiếp trên Snow devices

AWS DataSync:
└── Snowcone tích hợp DataSync để transfer qua mạng sau khi về AWS

AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa):
└── Quản lý encryption keys cho toàn bộ Snow Family

AWS IAM (Identity and Access Management — Quản Lý Danh Tính Truy Cập):
└── Kiểm soát quyền truy cập job và thiết bị

AWS CloudTrail (Nhật Ký Kiểm Toán):
└── Ghi lại tất cả API calls liên quan đến Snow jobs
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q1: Snow Family là gì và tại sao cần nó?**

> Snow Family là bộ thiết bị vật lý của AWS để di chuyển dữ liệu offline. Cần thiết khi dữ liệu quá lớn để chuyển qua mạng hiệu quả — quy tắc thực tế: nếu upload mất hơn 1 tuần, hãy dùng Snow Family. Ví dụ: 100 TB qua đường 1 Gbps mất ~9 ngày lý thuyết, thực tế còn lâu hơn và ảnh hưởng đến production traffic.

**Q2: Phân biệt 3 thiết bị Snow Family?**

> - **Snowcone**: Nhỏ nhất (2.1 kg), 8-14 TB, dùng ở vùng xa/thực địa, có thể chạy bằng pin
> - **Snowball Edge**: Dòng chính, 2 loại: Storage (80 TB) và Compute (28/42 TB + GPU), dùng cho di chuyển dữ liệu lớn và edge computing công nghiệp
> - **Snowmobile**: Xe tải container 100 PB, dùng cho exabyte-scale migration (> 10 PB)

**Q3: Dữ liệu được bảo mật thế nào trên Snow devices?**

> Mã hóa AES-256 end-to-end, khóa mã hóa quản lý bởi KMS (bạn sở hữu CMK). Phần cứng có TPM để phát hiện giả mạo vật lý. Sau khi import, AWS thực hiện secure erase theo NIST 800-88.

**Q4: Sau khi Snow device về AWS, dữ liệu đi đâu?**

> Dữ liệu được import vào S3 bucket mà bạn đã chỉ định khi đặt hàng. Từ S3, bạn có thể dùng lifecycle policies để chuyển sang Glacier, hoặc dùng cho các dịch vụ khác (EC2, EMR, Athena, v.v.).

### Câu Hỏi Tình Huống

**Q5: Khách hàng có 500 TB cần chuyển lên AWS, băng thông 100 Mbps, thời hạn 2 tháng. Bạn đề xuất gì?**

> Dùng Snowball Edge Storage Optimized. 500 TB ÷ 80 TB/thiết bị = ~7 thiết bị. Có thể xử lý song song: đặt 3-4 thiết bị cùng lúc, copy xong gửi đi, AWS giao thiết bị mới. Tổng thời gian: 4-6 tuần — phù hợp với deadline.
>
> Qua mạng 100 Mbps: 500 TB = ~500,000 GB ÷ 0.0125 GB/s ≈ 40M giây ≈ 463 ngày. Không khả thi.

**Q6: Công ty muốn thu thập sensor data từ giàn khoan dầu ngoài biển, không có internet. Giải pháp?**

> AWS Snowcone là lựa chọn tốt nhất:
> - Nhỏ gọn (2.1 kg), chịu điều kiện khắc nghiệt
> - Chạy được bằng pin hoặc USB-C
> - Có thể chạy edge computing (xử lý dữ liệu tại chỗ)
> - Khi tàu vào bờ, gửi Snowcone về AWS hoặc upload qua DataSync nếu có kết nối

**Q7: Khi nào dùng Snowmobile thay vì nhiều Snowball Edge?**

> - Dùng Snowmobile khi > 10 PB
> - Nếu < 10 PB: nhiều Snowball Edge (có thể cluster tối đa 15 nodes = ~1.2 PB)
> - Snowmobile có tốc độ transfer cao hơn (đến 1 Tbps), phù hợp datacenter consolidation quy mô lớn
> - Snowmobile cần cơ sở hạ tầng điện, mạng đặc biệt tại datacenter

**Q8: Snow Family khác DataSync như thế nào?**

> | Tiêu Chí | Snow Family | DataSync |
> |---|---|---|
> | Phương thức | Offline (vật lý) | Online (qua mạng) |
> | Phù hợp | > 10 TB, băng thông yếu | < 100 TB, băng thông tốt |
> | Thời gian | Vài ngày đến vài tuần | Vài giờ đến vài ngày |
> | Real-time | ✗ | ✅ (sync liên tục) |
> | Edge compute | ✅ (Snowball/Snowcone) | ✗ |

---

## 🗺️ Điều Hướng

- [1-snowcone.md](./1-snowcone.md) — Snowcone: 8-14 TB, edge computing nhỏ gọn
- [2-snowball-edge.md](./2-snowball-edge.md) — Snowball Edge: Storage vs Compute Optimized
- [3-snowmobile.md](./3-snowmobile.md) — Snowmobile: 100 PB, exabyte migration

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Thuộc Module:** 06-snow-family (AWS Migration & Transfer Services)
