# AWS Snowcone — Thiết Bị Nhỏ Gọn Nhất Cho Edge & Migration

> **AWS Snowcone** là thiết bị nhỏ gọn nhất trong AWS Snow Family — nặng chỉ 2.1 kg, bền bỉ với môi trường khắc nghiệt, có thể chạy bằng pin hoặc nguồn USB-C. Snowcone phục vụ hai mục đích chính: **di chuyển dữ liệu ngoại tuyến** (offline data migration) lên AWS và **edge computing** (tính toán tại điểm biên) tại những nơi không có hoặc có ít kết nối internet.

## 📚 Mục Lục (Table of Contents)

1. [Snowcone Là Gì? Khi Nào Dùng?](#snowcone-là-gì-khi-nào-dùng)
2. [Thông Số Kỹ Thuật Chi Tiết](#thông-số-kỹ-thuật-chi-tiết)
3. [Hai Phiên Bản: HDD vs SSD](#hai-phiên-bản-hdd-vs-ssd)
4. [Edge Computing Trên Snowcone](#edge-computing-trên-snowcone)
5. [Quy Trình Di Chuyển Dữ Liệu](#quy-trình-di-chuyển-dữ-liệu)
6. [Tích Hợp DataSync: Online Transfer Sau Khi Về AWS](#tích-hợp-datasync-online-transfer-sau-khi-về-aws)
7. [AWS OpsHub Trên Snowcone](#aws-opshub-trên-snowcone)
8. [Bảo Mật](#bảo-mật)
9. [Giá Và Chi Phí](#giá-và-chi-phí)
10. [So Sánh Với Snowball Edge](#so-sánh-với-snowball-edge)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Snowcone Là Gì? Khi Nào Dùng?

### Tổng Quan

```
AWS Snowcone = Snow Family nhỏ nhất
               Thiết bị bền bỉ + edge compute + offline data migration

Kích thước vật lý: 227 × 148 × 82 mm (nhỏ hơn hộp cơm trưa)
Trọng lượng:      2.1 kg
Nguồn điện:       USB-C (có thể dùng pin laptop hoặc power bank)
Chứng nhận:       MIL-STD-810H (tiêu chuẩn quân sự cho độ bền)
```

### Khi Nào Dùng Snowcone?

```
✅ PHÙ HỢP:

1. Thu thập dữ liệu thực địa (field data collection):
   ├── Sensor data từ giàn khoan dầu, mỏ khai thác, nông trại thông minh
   ├── Video surveillance tại vùng xa không có internet
   └── Scientific data collection (thu thập dữ liệu khoa học) ngoài thực địa

2. Edge computing ở vùng xa (remote edge computing):
   ├── Phân tích dữ liệu ngay tại chỗ (không cần gửi về datacenter)
   ├── ML inference (dự đoán bằng machine learning) tại thực địa
   └── Industrial IoT processing (xử lý IoT công nghiệp)

3. Di chuyển dữ liệu nhỏ (< 14 TB):
   ├── Branch office migration (di chuyển văn phòng chi nhánh)
   ├── Small datacenter consolidation (gộp datacenter nhỏ)
   └── Backup archive migration (di chuyển kho backup)

4. Môi trường không có internet ổn định:
   ├── Tàu chiến, tàu nghiên cứu, thuyền đánh cá lớn
   ├── Địa điểm xây dựng xa xôi
   └── Disaster response (ứng phó thảm họa) — mất kết nối mạng

❌ KHÔNG PHÙ HỢP:

├── Dữ liệu > 14 TB → dùng Snowball Edge
├── Cần GPU mạnh → Snowball Edge Compute Optimized có NVIDIA V100
├── Cần EC2 instances lớn → Snowcone chỉ có 2 vCPU, 4 GB RAM
└── Băng thông internet tốt → dùng DataSync trực tiếp (nhanh hơn)
```

---

## 📐 Thông Số Kỹ Thuật Chi Tiết

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS SNOWCONE — SPECS                          │
├─────────────────────────────┬───────────────────────────────────┤
│ Thuộc Tính                  │ Giá Trị                           │
├─────────────────────────────┼───────────────────────────────────┤
│ Kích thước (L × W × H)      │ 227 × 148 × 82 mm                 │
│ Trọng lượng                 │ 2.1 kg                            │
│ Nguồn điện                  │ USB-C 45W (điện thoại, laptop,    │
│                             │ power bank, pin xe ô tô)          │
│ Nhiệt độ hoạt động          │ -20°C đến 45°C                    │
│ Chứng nhận bền bỉ           │ MIL-STD-810H                      │
├─────────────────────────────┼───────────────────────────────────┤
│ vCPU (bộ xử lý ảo)          │ 2 vCPUs (Intel Xeon)              │
│ RAM (bộ nhớ)                │ 4 GB                              │
│ GPU (bộ xử lý đồ họa)       │ Không có                          │
├─────────────────────────────┼───────────────────────────────────┤
│ Dung lượng HDD (usable)     │ 8 TB (HDD model)                  │
│ Dung lượng SSD (usable)     │ 14 TB (SSD model)                 │
├─────────────────────────────┼───────────────────────────────────┤
│ Network interface           │ 2 × 10/25 GbE (SFP28)            │
│                             │ + WiFi (một số phiên bản)         │
├─────────────────────────────┼───────────────────────────────────┤
│ EC2 Instance types hỗ trợ   │ sbe-c.small, sbe-c.medium         │
│ Lambda Functions            │ ✅ Hỗ trợ                         │
├─────────────────────────────┼───────────────────────────────────┤
│ Mã hóa                      │ AES-256 (256-bit)                 │
│ TPM                         │ ✅ Trusted Platform Module        │
│ Thời gian thuê tối đa        │ 360 ngày/lần                      │
└─────────────────────────────┴───────────────────────────────────┘
```

---

## 💾 Hai Phiên Bản: HDD vs SSD

```
┌─────────────────────────────────────────────────────────────────┐
│              SNOWCONE HDD vs SNOWCONE SSD                       │
├────────────────────────┬────────────────┬───────────────────────┤
│ Tiêu Chí               │ Snowcone HDD   │ Snowcone SSD          │
├────────────────────────┼────────────────┼───────────────────────┤
│ Dung lượng usable      │ 8 TB           │ 14 TB                 │
│ Loại ổ đĩa             │ HDD (cơ học)   │ SSD (bán dẫn)         │
│ Tốc độ đọc/ghi         │ Chậm hơn       │ Nhanh hơn (~5x)       │
│ Chịu rung động         │ Kém hơn        │ Tốt hơn               │
│ Chi phí                │ Thấp hơn       │ Cao hơn               │
├────────────────────────┼────────────────┼───────────────────────┤
│ Phù hợp nhất           │ Thu thập dữ    │ Field analytics       │
│                        │ liệu tĩnh,     │ (phân tích tại chỗ)   │
│                        │ môi trường ổn  │ môi trường có rung    │
│                        │ định           │ động (xe, máy bay)    │
└────────────────────────┴────────────────┴───────────────────────┘

Lời khuyên:
├── Môi trường rung lắc (xe tải, máy bay, tàu) → Chọn SSD
├── Chỉ cần thu thập và gửi về → HDD đủ dùng và rẻ hơn
└── Cần xử lý dữ liệu tại chỗ nhiều → SSD (I/O nhanh hơn)
```

---

## 💻 Edge Computing Trên Snowcone

### Chạy EC2 Instances Tại Điểm Biên

```
Snowcone hỗ trợ chạy EC2 instances cục bộ (locally) — không cần AWS Region.

Instance types được hỗ trợ:
├── sbe-c.small:  1 vCPU, 1 GB RAM
└── sbe-c.medium: 2 vCPU, 4 GB RAM (toàn bộ tài nguyên thiết bị)

Nguồn AMI (Amazon Machine Image — Ảnh Máy):
└── Import AMI từ AWS Region vào thiết bị trước khi triển khai

Ví dụ ứng dụng edge trên Snowcone:
├── 1. Anomaly detection (phát hiện bất thường) từ sensor data
├── 2. Video thumbnail generation (tạo ảnh thumbnail từ video)
├── 3. Data compression/preprocessing trước khi gửi về cloud
├── 4. Local database (SQLite, PostgreSQL nhỏ) cho field workers
└── 5. Lightweight web server phục vụ UI cho nhân viên thực địa
```

### Chạy Lambda Functions

```
AWS Lambda on Snowcone (Greengrass V2):
├── Triển khai Lambda functions xuống thiết bị
├── Trigger (kích hoạt) theo sự kiện: file mới, timer, MQTT message
├── Xử lý dữ liệu ngay khi thu thập (real-time preprocessing)
└── Đồng bộ kết quả lên AWS khi có kết nối

Use case điển hình:
Event: File sensor được ghi vào thiết bị
  → Lambda trigger
  → Xử lý, lọc, nén dữ liệu
  → Lưu kết quả đã xử lý (nhỏ hơn nhiều)
  → Gửi lên S3 khi có kết nối
```

### AWS IoT Greengrass Tích Hợp

```
AWS IoT Greengrass (Cỏ Xanh IoT) — Runtime cho edge computing:
├── Chạy trên Snowcone
├── Quản lý lifecycle (vòng đời) của Lambda và container components
├── Sync messages giữa devices và AWS IoT Core
└── Offline-first: tiếp tục hoạt động khi mất kết nối AWS

Kiến trúc điển hình:
[Sensors/Cameras] → [Snowcone + Greengrass] → [Lambda xử lý]
                                                      │
                    (khi có internet)                 ▼
                              [S3 / IoT Core / DynamoDB]
```

---

## 🔄 Quy Trình Di Chuyển Dữ Liệu

### Offline Transfer (Gửi Thiết Bị Về AWS)

```
BƯỚC 1: ĐẶT JOB TRONG AWS CONSOLE
├── Chọn job type: "Import into Amazon S3"
├── Cấu hình S3 bucket đích
├── Chọn model (HDD 8 TB hay SSD 14 TB)
└── Nhập địa chỉ giao hàng

BƯỚC 2: NHẬN THIẾT BỊ (2-7 ngày)
├── AWS giao Snowcone đến địa chỉ bạn
└── Kèm: cáp USB-C, cáp RJ45, tài liệu hướng dẫn

BƯỚC 3: UNLOCK VÀ KẾT NỐI
├── Cài AWS OpsHub (GUI) hoặc Snowball client (CLI)
├── Unlock thiết bị bằng credentials từ AWS Console
└── Kết nối thiết bị vào mạng nội bộ (LAN/WiFi)

BƯỚC 4: COPY DỮ LIỆU
├── Qua NFS mount:
│     mount -t nfs <Snowcone-IP>:/buckets/s3/<bucket-name> /mnt/snowcone
│     cp -r /data/to/migrate/* /mnt/snowcone/
│
├── Qua AWS OpsHub:
│     Drag-and-drop files vào thiết bị
│
└── Tốc độ: tối đa 1-10 Gbps tùy network và storage type

BƯỚC 5: GỬI TRẢ VỀ AWS
├── Power off thiết bị qua OpsHub
├── E-ink label tự cập nhật địa chỉ AWS facility
└── Giao cho UPS/DHL

BƯỚC 6: AWS IMPORT VÀO S3
├── AWS nhận thiết bị, giải mã, import dữ liệu vào S3
├── Thời gian import: thường 1-3 ngày làm việc
└── Nhận email thông báo khi hoàn tất
```

---

## 🌐 Tích Hợp DataSync: Online Transfer Sau Khi Về AWS

### Cơ Chế Hybrid: Offline + Online

```
Snowcone là thiết bị Snow Family DUY NHẤT tích hợp DataSync agent sẵn có.

Điều này cho phép 2 luồng làm việc linh hoạt:

LUỒNG 1: Pure Offline (Ngoại Tuyến Hoàn Toàn)
[Thu thập dữ liệu] → [Copy vào Snowcone] → [Gửi về AWS] → [Import S3]
Dùng khi: Không bao giờ có internet

LUỒNG 2: Hybrid — Offline Thu Thập, Online Đồng Bộ
[Thu thập dữ liệu vào Snowcone] →
  Khi có internet:
  [DataSync agent trên Snowcone] → [Sync trực tiếp lên S3]
Dùng khi: Thu thập dữ liệu ở nơi không có internet
         Về đến nơi có WiFi/LAN → tự động sync

Ví dụ thực tế:
├── Drone survey: bay trên cánh đồng không có internet
├── Dữ liệu ảnh chụp lưu vào Snowcone
├── Về trại (camp) có WiFi vào buổi tối
└── DataSync tự động sync tất cả ảnh lên S3

Không cần gửi thiết bị về AWS — tiết kiệm thời gian và chi phí ship!
```

### Cấu Hình DataSync Trên Snowcone

```bash
# Kích hoạt DataSync agent từ AWS Console:
# 1. Mở Snow Family job trong Console
# 2. Enable DataSync agent
# 3. AWS tự cài agent trên Snowcone
# 4. Từ DataSync Console: tạo location trỏ vào Snowcone
# 5. Tạo task: Snowcone location → S3 location
# 6. Chạy task khi có kết nối

# Không cần cài đặt thêm phần mềm trên thiết bị
```

---

## 🖥️ AWS OpsHub Trên Snowcone

```
OpsHub = GUI application (ứng dụng giao diện đồ họa) cho Snow Family

Cài đặt trên: Windows, macOS
Kết nối: Cùng mạng LAN với Snowcone

Tính năng chính:
┌─────────────────────────────────────────────────────────┐
│  Tab                 │  Chức Năng                        │
├──────────────────────┼───────────────────────────────────┤
│  Device Status       │  Trạng thái thiết bị, nhiệt độ,   │
│  (Trạng Thái)        │  dung lượng còn lại               │
├──────────────────────┼───────────────────────────────────┤
│  File Transfer       │  Copy files bằng drag-and-drop    │
│  (Chuyển File)       │  hoặc nhập đường dẫn              │
├──────────────────────┼───────────────────────────────────┤
│  Compute             │  Quản lý EC2 instances, start/stop│
│  (Tính Toán)         │  instances, xem logs              │
├──────────────────────┼───────────────────────────────────┤
│  Network             │  Cấu hình IP, xem kết nối mạng    │
│  (Mạng)              │                                   │
├──────────────────────┼───────────────────────────────────┤
│  Alarms & Metrics    │  Cảnh báo dung lượng, CPU, nhiệt  │
│  (Cảnh Báo)          │  độ vượt ngưỡng                   │
└──────────────────────┴───────────────────────────────────┘

Trước OpsHub (trước 2020): Phải dùng Snowball client CLI phức tạp
Sau OpsHub:              Giao diện thân thiện, kéo thả đơn giản
```

---

## 🔒 Bảo Mật

### Mã Hóa Và Bảo Vệ Vật Lý

```
MÃ HÓA DỮ LIỆU:
├── Thuật toán: AES-256 (256-bit Advanced Encryption Standard)
├── Khóa: Quản lý bởi AWS KMS (Key Management Service)
│   ├── Bạn có thể dùng AWS-managed key
│   └── Hoặc tạo CMK (Customer Master Key — Khóa Chính Khách Hàng) riêng
├── Mã hóa tự động: Tất cả data ghi vào thiết bị đều được mã hóa
└── AWS không bao giờ có quyền truy cập vào CMK của bạn

BẢO VỆ VẬT LÝ:
├── TPM — Trusted Platform Module (Module Bảo Mật Tin Cậy):
│   └── Chip phần cứng xác minh thiết bị chưa bị giả mạo
├── Tamper-evident design (thiết kế phát hiện can thiệp vật lý):
│   └── Nếu thiết bị bị mở bung, TPM phát hiện và khóa thiết bị
├── No serviceable parts (không có bộ phận có thể tháo ra):
│   └── Người dùng không thể tự ý tháo lắp bên trong
└── MIL-STD-810H: Chịu va đập, rung, nhiệt độ cực đoan

SAU KHI IMPORT HOÀN TẤT:
└── AWS thực hiện secure erase (xóa an toàn) theo NIST 800-88
    (National Institute of Standards and Technology — Viện Tiêu Chuẩn Quốc Gia Mỹ)

KIỂM SOÁT TRUY CẬP:
├── Unlock code: Mã riêng biệt gửi qua email, chỉ bạn nhận được
├── Manifest file: Xác thực thiết bị đúng là thiết bị được giao
└── CloudTrail logging: Tất cả thao tác được ghi lại
```

---

## 💰 Giá Và Chi Phí

```
Mô hình tính phí Snowcone (on-demand, không trả trước):

PHẦN 1: PHÍ THIẾT BỊ (Device Fee)
├── Snowcone HDD (8 TB): ~$60/lần dùng (tính theo job)
└── Snowcone SSD (14 TB): ~$90/lần dùng

PHẦN 2: PHÍ VẬN CHUYỂN (Shipping Fee)
├── Phụ thuộc vào khu vực địa lý
└── Thường $20-50 một chiều

PHẦN 3: PHÍ CHUYỂN DỮ LIỆU VÀO S3 (Data Transfer In)
└── MIỄN PHÍ — Import vào S3 không mất phí transfer

PHẦN 4: STORAGE S3 (nếu lưu trữ lâu dài)
└── Tính theo standard S3 pricing (~$0.023/GB/tháng)

SO SÁNH VỚI NETWORK TRANSFER (Internet Upload):
├── 8 TB qua internet 100 Mbps: ~8 ngày + tốn băng thông
└── 8 TB qua Snowcone: ~2-3 ngày (ship) + không tốn băng thông

Snowcone thường rẻ hơn và nhanh hơn khi bandwidth hạn chế
```

---

## 📊 So Sánh Với Snowball Edge

```
┌─────────────────────────────────────────────────────────────────┐
│              SNOWCONE vs SNOWBALL EDGE                          │
├────────────────────────┬───────────────┬────────────────────────┤
│ Tiêu Chí               │ Snowcone      │ Snowball Edge (Storage)│
├────────────────────────┼───────────────┼────────────────────────┤
│ Dung lượng             │ 8-14 TB       │ 80 TB                  │
│ vCPU                   │ 2             │ 40                     │
│ RAM                    │ 4 GB          │ 80 GB                  │
│ GPU                    │ Không         │ Không (Compute có GPU) │
│ Kích thước             │ Hộp cơm       │ Vali nhỏ               │
│ Trọng lượng            │ 2.1 kg        │ ~22 kg                 │
│ Pin/Battery            │ ✅ USB-C      │ ✗ (cần ổ điện)         │
│ DataSync tích hợp      │ ✅            │ ✗                      │
│ WiFi                   │ ✅ (một số)   │ ✗                      │
│ Clustering             │ ✗             │ ✅ (tối đa 15 nodes)   │
│ Giá                    │ Thấp hơn      │ Cao hơn                │
├────────────────────────┼───────────────┼────────────────────────┤
│ Dùng khi               │ < 14 TB,      │ 14-80 TB,              │
│                        │ vùng xa,      │ edge computing         │
│                        │ cần di động   │ công nghiệp nặng       │
└────────────────────────┴───────────────┴────────────────────────┘
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q1: AWS Snowcone được dùng cho mục đích gì?**

> Snowcone có hai mục đích chính:
> 1. **Offline data migration** (di chuyển dữ liệu ngoại tuyến): Copy đến 14 TB lên S3 bằng cách gửi thiết bị vật lý về AWS — phù hợp khi băng thông mạng hạn chế.
> 2. **Edge computing** (tính toán tại điểm biên): Chạy EC2 instances và Lambda functions tại chỗ, xử lý dữ liệu ở môi trường không có internet hoặc kết nối yếu (giàn khoan, tàu, địa điểm xây dựng).

**Q2: Snowcone khác gì so với Snowball Edge?**

> Snowcone nhỏ hơn nhiều (2.1 kg vs 22 kg), dung lượng thấp hơn (14 TB vs 80 TB), tài nguyên tính toán ít hơn (2 vCPU, 4 GB RAM vs 40 vCPU, 80 GB RAM). Ưu điểm riêng của Snowcone: chạy được bằng pin USB-C, tích hợp DataSync agent sẵn (Snowball Edge không có), nhỏ gọn phù hợp môi trường hạn chế không gian.

**Q3: DataSync tích hợp trên Snowcone có ý nghĩa gì?**

> Cho phép Snowcone hoạt động theo mô hình hybrid: thu thập dữ liệu offline khi không có internet, sau đó khi có kết nối, DataSync agent tự động sync dữ liệu lên S3 — không cần gửi thiết bị về AWS. Đây là tính năng độc đáo của Snowcone, không có trên Snowball Edge hay Snowmobile.

**Q4: Snowcone phù hợp với use case nào nhất?**

> - Drone survey, field research (nghiên cứu thực địa)
> - IoT data collection tại vùng sâu vùng xa
> - Tàu biển (thu thập dữ liệu khi ra khơi, sync khi về cảng)
> - Disaster response (ứng phó thảm họa) khi mạng bị gián đoạn
> - Small branch office migration (< 14 TB)

**Q5: Snowcone có thể chạy Docker containers không?**

> Gián tiếp — Snowcone chạy EC2 instances (sbe-c.small, sbe-c.medium), và các instances này có thể chạy Docker. Tuy nhiên với chỉ 2 vCPU và 4 GB RAM, chỉ phù hợp cho containers nhẹ (lightweight containers).

---

## 🗺️ Điều Hướng

- [README.md](./README.md) — Tổng quan & so sánh Snow Family
- [2-snowball-edge.md](./2-snowball-edge.md) — Snowball Edge: Storage vs Compute Optimized
- [3-snowmobile.md](./3-snowmobile.md) — Snowmobile: 100 PB, exabyte migration

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Thuộc Module:** 06-snow-family (AWS Migration & Transfer Services)
