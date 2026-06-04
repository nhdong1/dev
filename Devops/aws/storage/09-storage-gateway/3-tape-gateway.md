# 📼 Tape Gateway — Thư Viện Băng Ảo (Virtual Tape Library)

> Tape Gateway (Cổng Băng Từ) là loại AWS Storage Gateway mô phỏng một VTL (Virtual Tape Library — Thư Viện Băng Ảo) cho phép phần mềm backup doanh nghiệp ghi dữ liệu lên "băng ảo" thực chất được lưu trên Amazon S3 và S3 Glacier, thay thế hoàn toàn băng từ vật lý (physical tape).

---

## 📚 Mục Lục

1. [Tại Sao Cần Thay Thế Tape Vật Lý](#tại-sao-cần-thay-thế)
2. [Kiến Trúc Tape Gateway](#kiến-trúc-tape-gateway)
3. [Virtual Tape — Băng Ảo](#virtual-tape)
4. [Vòng Đời Băng Ảo](#vòng-đời-băng-ảo)
5. [Tích Hợp Backup Software](#tích-hợp-backup-software)
6. [Use Cases Điển Hình](#use-cases-điển-hình)
7. [Hiệu Suất](#hiệu-suất)
8. [Compliance và Retention](#compliance-và-retention)
9. [Chi Phí](#chi-phí)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Thay Thế

### Vấn Đề Của Tape Vật Lý

```
Chi phí:
├── Tape drive (đầu đọc băng): $10,000–$50,000/thiết bị
├── Tape library (tủ chứa băng): $100,000–$500,000
├── Băng LTO-8: $20–30/băng, mỗi hệ thống cần hàng nghìn băng
├── Offsite storage (lưu trữ ngoài site): $500–2,000/tháng
└── Nhân sự: 1–2 FTE (Full-Time Employee) quản lý tape

Vận hành:
├── Thay băng thủ công hàng ngày/tuần
├── Tracking inventory (quản lý kho) phức tạp
├── Băng hỏng, mất, đọc lỗi → mất dữ liệu backup
├── Restore chậm: tìm băng → đưa vào drive → mount → restore
└── RTO (Recovery Time Objective) thường tính bằng ngày

Rủi ro:
├── Băng vật lý: degradation (xuống cấp) theo thời gian
├── Offsite transportation: băng có thể bị mất hoặc đánh cắp
└── Disaster (hỏa hoạn, lũ lụt): mất cả kho băng onsite
```

### Tape Gateway Giải Quyết Gì

```
Thay thế hoàn toàn:
├── Tape drive vật lý → VTL interface (iSCSI) 
├── Băng vật lý → Virtual Tape (lưu trên S3/Glacier)
├── Tape library vật lý → Virtual Tape Shelf (VTS)
├── Offsite transport → automatic archive to Glacier
└── Manual tape management → AWS Console

Kết quả:
├── Không cần hardware tape → tiết kiệm $100,000+
├── Không cần offsite transport
├── Restore nhanh hơn (minutes thay vì days)
└── Durability: S3 = 99.999999999% so với tape = ~99.9%
```

---

## Kiến Trúc Tape Gateway

```
Backup Software (Veeam, Veritas NetBackup, Commvault, etc.)
        │
        │  VTL Interface (iSCSI)
        │  (Thư Viện Băng Ảo qua giao thức iSCSI)
        ▼
┌─────────────────────────────────────────────────────────────┐
│           Tape Gateway (VM hoặc Hardware)                   │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Virtual Tape Library (VTL)                     │  │
│  │                                                        │  │
│  │  ┌──────────────────┐   ┌──────────────────────────┐  │  │
│  │  │  Virtual Tape    │   │  Medium Changer           │  │  │
│  │  │  Drive (×10 max) │   │  (Cơ Chế Thay Đổi Phương │  │  │
│  │  │  (Ổ Đọc Băng Ảo) │   │   Tiện — robot arm)       │  │  │
│  │  └──────────────────┘   └──────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Upload Buffer                            │  │
│  │    (Lưu tạm trước khi upload lên S3)                  │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS/TLS
                    ┌──────▼──────────────────────┐
                    │         Amazon S3            │
                    │  Virtual Tape Library (VTL)  │
                    │  (Băng đang hoạt động)       │
                    └──────────────────────────────┘
                           │
                           │ Archive (Lưu trữ lâu dài)
                           ▼
                    ┌──────────────────────────────┐
                    │      S3 Glacier /             │
                    │  S3 Glacier Deep Archive      │
                    │  Virtual Tape Shelf (VTS)     │
                    │  (Băng đã lưu trữ)            │
                    └──────────────────────────────┘
```

### Thành Phần VTL

| Thành Phần | Vật Lý | Ảo (Tape Gateway) |
|------------|--------|-------------------|
| **Tape Drive** | Đầu đọc băng vật lý | iSCSI device (tối đa 10 drives) |
| **Tape Cartridge** | Băng LTO vật lý | Virtual Tape (100GB – 5TB/băng) |
| **Tape Library** | Tủ chứa băng vật lý | VTL trong S3 |
| **Tape Shelf** | Kho offsite | VTS trong S3 Glacier |
| **Medium Changer** | Robot arm trong tủ băng | Phần mềm tự động |

---

## Virtual Tape — Băng Ảo

### Đặc Điểm

```
Kích thước: 100 GB đến 5 TB mỗi băng ảo
Tổng: Tối đa 1 PB trên một gateway
Số lượng: Tối đa 1,500 virtual tapes per gateway

Lưu trữ:
├── Active tapes (đang dùng): lưu trên S3
└── Archived tapes (đã lưu trữ): lưu trên S3 Glacier/Deep Archive

Trạng thái:
├── AVAILABLE: sẵn sàng trong VTL
├── IN-USE: backup software đang ghi
├── RETRIEVED: vừa restore từ Glacier
└── ARCHIVING: đang chuyển sang Glacier
```

### Tạo Virtual Tape

```bash
# AWS CLI tạo virtual tape
aws storagegateway create-tapes \
  --gateway-arn arn:aws:storagegateway:region:account:gateway/sgw-xxx \
  --tape-size-in-bytes 107374182400 \  # 100GB
  --num-tapes-to-create 5 \
  --tape-barcode-prefix MY \
  --pool-id pool-xxx

# Tape barcode: MY000001, MY000002, ... (phần mềm backup nhận dạng qua barcode)
```

---

## Vòng Đời Băng Ảo

### Lifecycle Hoàn Chỉnh

```
1. CREATE (Tạo)
   AWS Console/CLI → tạo virtual tape → AVAILABLE trong VTL

2. USE (Sử Dụng)
   Backup software phát hiện tape → mount → ghi data backup
   Tape status: IN-USE
   Data lưu: Gateway upload buffer → S3

3. EJECT (Đẩy Ra)
   Backup job hoàn thành → backup software eject tape
   Tape trở về VTL với status AVAILABLE

4. ARCHIVE (Lưu Trữ)
   Admin xuất tape sang Tape Shelf:
   VTL (S3) → VTS (S3 Glacier/Deep Archive)
   Lý do: tiết kiệm chi phí, compliance retention

5. RETRIEVE (Khôi Phục)
   Khi cần restore: retrieve từ Glacier → S3
   Thời gian: 3-5 giờ (Standard) hoặc 1-5 phút (Expedited)
   Sau đó backup software có thể mount và restore data

6. DELETE (Xóa)
   Sau khi hết retention period: xóa virtual tape
   Giải phóng S3/Glacier storage
```

### Archive Pools (Nhóm Lưu Trữ)

```
Tape Gateway hỗ trợ 2 loại pool để archive:

Glacier Pool:
├── Backend: S3 Glacier Flexible Retrieval
├── Giá: $0.004/GB/tháng
├── Retrieval: 3-5 giờ (Standard), 1-5 phút (Expedited)
└── Use case: backup cần restore trong vài giờ

Deep Archive Pool:
├── Backend: S3 Glacier Deep Archive
├── Giá: $0.00099/GB/tháng (rẻ nhất AWS)
├── Retrieval: 12 giờ (Standard), 48 giờ (Bulk)
└── Use case: long-term compliance archive (7+ năm)
```

---

## Tích Hợp Backup Software

### Phần Mềm Backup Hỗ Trợ

```
Tương thích với hầu hết phần mềm backup enterprise:
├── Veritas NetBackup
├── Veritas Backup Exec
├── Veeam Backup & Replication
├── Commvault
├── IBM Spectrum Protect (TSM)
├── Dell EMC NetWorker
└── Arcserve UDP
```

### Cấu Hình Veritas NetBackup

```
Bước 1: Add Media Server
NetBackup Administration Console
→ Media and Device Management
→ Devices → Tape Drives
→ Add iSCSI target: <gateway-ip>:3260

Bước 2: Scan for Devices
Chờ NetBackup discover:
- Medium Changer (robot)
- Tape Drives (10 units)
- Virtual Tapes (barcodes: MY000001–MY001500)

Bước 3: Create Storage Unit
Storage Type: Media Manager
Media Server: <media-server>
Density: 1/2-in. Cartridge (chọn phù hợp)

Bước 4: Create Backup Policy
Policy Type: Standard
Schedule: Daily Full + Weekly Incremental
Storage Unit: VTL-Storage-Unit

# Backup bắt đầu chạy như với tape vật lý thực sự
```

### Cấu Hình Veeam

```
Veeam → Backup Infrastructure → Add Backup Repository
→ Tape → Add Tape Server
→ Target: <gateway-ip>

Veeam tự động:
├── Discover VTL với 10 tape drives
├── Inventory tất cả virtual tapes
└── Assign tapes cho backup jobs

Lưu ý quan trọng:
→ Enable "Use per-machine backup files" để tránh full restore
→ Set tape pool với barcodes matching gateway tapes
```

---

## Use Cases Điển Hình

### 1. Thay Thế Tape Library Cũ

```
Before:
├── StorageTek L700 tape library, 700 slots
├── 4 LTO-6 drives
├── 12TB/tuần backup
├── Offsite truck pickup mỗi Thứ Sáu
└── Chi phí: $15,000/tháng (amortization + ops)

After với Tape Gateway:
├── Tape Gateway VM (4 vCPU, 16GB RAM)
├── Virtual tapes 2.5TB mỗi băng
├── Upload lên S3 qua Direct Connect 1Gbps
├── Tự động archive sang Glacier sau 30 ngày
└── Chi phí: ~$3,000/tháng → tiết kiệm 80%
```

### 2. Regulatory Compliance (Tuân Thủ Quy Định)

```
Yêu cầu: FINRA (Financial Industry Regulatory Authority) 
yêu cầu lưu giữ dữ liệu giao dịch tài chính 7 năm

Setup:
├── Veeam backup → Tape Gateway → VTL
├── Sau 30 ngày: archive sang Glacier Deep Archive
├── S3 Object Lock trên underlying storage
└── Glacier Vault Lock: không xóa được trong 7 năm

Chi phí 1PB lưu trữ 7 năm:
Tape vật lý: ~$50,000/năm × 7 = $350,000
Glacier Deep Archive: $0.00099 × 1,000,000 × 12 × 7 = ~$83,000
→ Tiết kiệm 76%
```

### 3. Backup-as-a-Service cho SMB

```
Scenario: MSP (Managed Service Provider — Nhà Cung Cấp Dịch Vụ Quản Lý)
cung cấp backup service cho 50 khách hàng SMB

Mỗi khách:
├── 1 Tape Gateway (VM) tại site khách hàng
├── Backup software (Veeam) kết nối local gateway
├── Data tự động lên S3 của MSP
└── Glacier archive cho long-term retention

MSP quản lý tập trung:
├── AWS console: monitor tất cả gateways
├── S3 buckets riêng per customer
└── Billing rõ ràng per GB
```

---

## Hiệu Suất

### Throughput (Thông Lượng)

```
Tape Gateway performance:
├── Tối đa 10 virtual tape drives đồng thời
├── Mỗi drive: ~40 MB/s upload speed
├── Tổng tối đa: ~400 MB/s aggregate

Bottleneck thường là:
├── Băng thông internet (100Mbps = 12.5 MB/s)
└── Direct Connect 1Gbps = 125 MB/s per drive → ideal

Sizing cho backup window:
12 giờ backup window × 40 MB/s = 1.7 TB per tape drive
10 drives × 1.7 TB = 17 TB/đêm
→ Đủ cho hầu hết SME (Small and Medium Enterprise)
```

### Upload Buffer

```
Upload Buffer (Bộ Đệm Upload):
├── Minimum: 150 GB
├── Khuyến nghị: 150% × daily backup size
└── Dùng SSD: tránh bottleneck khi nhiều drives ghi song song

Ví dụ:
Daily backup: 2 TB
Upload buffer nên: 3 TB SSD
```

---

## Compliance và Retention

### WORM (Write Once Read Many) cho Virtual Tapes

```
Tape Gateway hỗ trợ WORM tapes:
├── Khi tạo virtual tape: đánh dấu là WORM
├── Sau khi ghi: không thể modify hoặc overwrite
└── Phù hợp: financial records, healthcare (HIPAA), legal

Cấu hình:
aws storagegateway create-tapes \
  --worm true \   # <-- WORM flag
  --pool-id pool-xxx \
  ...
```

### Retention Lock

```
Kết hợp với S3 Object Lock:
├── Virtual tape lưu trên S3 với Object Lock
├── Glacier Vault Lock cho archived tapes
└── Không ai (kể cả admin) có thể xóa trước thời hạn

Pipeline:
Create WORM tape → Backup job ghi → Archive to Glacier
→ Glacier Vault Lock: retained 7 năm → auto-expire
```

---

## Chi Phí

### Cấu Trúc Phí

```
1. Data written to S3 VTL:
   $0.01/GB data written (giống các gateway khác)

2. Virtual Tape Storage (S3):
   S3 Standard: $0.023/GB/tháng
   (nhưng thường archive sang Glacier sớm)

3. Archived Tape Storage:
   Glacier: $0.004/GB/tháng
   Deep Archive: $0.00099/GB/tháng

4. Retrieval từ Glacier:
   Standard (3-5h): $0.01/GB
   Expedited (1-5min): $0.03/GB
   Bulk (5-12h): $0.0025/GB

5. Data Transfer Out:
   Từ S3 về on-prem: $0.09/GB
```

### So Sánh Chi Phí Thực Tế

```
Scenario: 50TB backup data, retain 5 năm

Physical Tape:
├── Hardware (amortized): $2,000/tháng
├── Tapes: 50TB × $20/tape = $1,000 (mua thêm hàng năm)
├── Offsite storage: $800/tháng
├── Staff: 0.5 FTE × $80,000 = $3,333/tháng
└── Total: ~$7,133/tháng = ~$85,600/năm

Tape Gateway + Glacier Deep Archive:
├── Gateway VM: ~$200/tháng
├── S3 VTL (active): 5TB × $0.023 = $115/tháng
├── Glacier Deep Archive: 45TB × $0.00099 × 1000 = $44/tháng
├── Data write: 50TB × $0.01 × 1000 = $500/tháng (new data)
└── Total: ~$859/tháng = ~$10,308/năm

Tiết kiệm: ~88% ($75,000+/năm)
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Tape Gateway là gì và tại sao dùng nó thay vì tape vật lý?**

> Tape Gateway cung cấp VTL (Virtual Tape Library) qua iSCSI, thay thế toàn bộ hạ tầng tape vật lý. Backup software không cần thay đổi gì vì vẫn thấy "tape drives" qua VTL interface. Lợi ích: không cần phần cứng đắt tiền, không cần offsite transport, tự động archive sang Glacier, chi phí giảm 80-90%, durability cao hơn nhiều.

**Q2: Phân biệt VTL và VTS trong Tape Gateway.**

> - **VTL (Virtual Tape Library)**: tapes đang hoạt động, lưu trên S3 Standard. Backup software có thể mount và dùng ngay.
> - **VTS (Virtual Tape Shelf)**: tapes đã archive (eject khỏi VTL), lưu trên S3 Glacier hoặc Deep Archive. Cần retrieve (3-5h Glacier, 12h Deep Archive) trước khi dùng lại.
> Analogie: VTL = kệ băng trong phòng → VTS = kho offsite.

**Q3: Làm sao restore dữ liệu từ archived tape?**

> 1. Console/CLI: retrieve virtual tape từ VTS về VTL
> 2. Chờ Glacier retrieve (Standard: 3-5 giờ, Expedited: 1-5 phút)
> 3. Tape chuyển từ ARCHIVING → RETRIEVED → AVAILABLE
> 4. Backup software mount tape, chạy restore job bình thường

**Q4: Tape Gateway hỗ trợ WORM không? Khi nào cần?**

> Có. Khi tạo virtual tape với flag WORM=true, tape không thể bị overwrite sau khi ghi. Kết hợp Glacier Vault Lock để lock retention policy. Cần cho: tài chính (FINRA 7 năm), y tế (HIPAA), pháp lý — nơi dữ liệu phải bất biến.

**Q5: Khác biệt giữa Tape Gateway và File/Volume Gateway?**

> - **Tape Gateway**: giao thức VTL/iSCSI, sequential write (ghi tuần tự như băng), backend là Glacier, use case là backup/archive
> - **File Gateway**: giao thức NFS/SMB, random access, backend là S3, use case là file server
> - **Volume Gateway**: giao thức iSCSI, block access, backend là S3 + EBS snapshots, use case là database/block storage

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
