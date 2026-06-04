# 💽 Volume Gateway — Lưu Trữ Khối iSCSI Lai với AWS

> Volume Gateway (Cổng Lưu Trữ Khối) cung cấp block storage (lưu trữ theo khối) qua giao thức iSCSI (Internet Small Computer System Interface — Giao Thức Lưu Trữ Khối Qua Mạng IP) cho ứng dụng on-premises, với dữ liệu được lưu trữ và backup trên Amazon S3. Có hai chế độ: **Cached Mode** (Chế Độ Cache) và **Stored Mode** (Chế Độ Lưu Cục Bộ).

---

## 📚 Mục Lục

1. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
2. [Cached Mode vs Stored Mode](#cached-mode-vs-stored-mode)
3. [iSCSI — Giao Thức Kết Nối](#iscsi)
4. [Snapshots và EBS Integration](#snapshots-và-ebs-integration)
5. [Use Cases Điển Hình](#use-cases-điển-hình)
6. [Cấu Hình và Triển Khai](#cấu-hình-và-triển-khai)
7. [Hiệu Suất](#hiệu-suất)
8. [Bảo Mật](#bảo-mật)
9. [Chi Phí](#chi-phí)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Tổng Quan

```
On-Premises Application
        │
        │  iSCSI (TCP port 3260)
        │  (Block I/O — đọc/ghi theo khối)
        ▼
┌─────────────────────────────────────────────────────────┐
│              Volume Gateway (VM/Hardware)                │
│                                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Volume (Block Device)                  │  │
│  │  ┌────────────────────┐ ┌──────────────────────┐  │  │
│  │  │   Upload Buffer    │ │    Cache / Primary   │  │  │
│  │  │  (Bộ Đệm Upload)   │ │    Storage           │  │  │
│  │  └────────────────────┘ └──────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS/TLS
                           ▼
                   ┌───────────────┐      ┌───────────────┐
                   │  Amazon S3   │ ───► │  EBS Snapshot  │
                   │  (Volume     │      │  (restore để  │
                   │   Storage)   │      │  tạo EBS vol) │
                   └───────────────┘      └───────────────┘
```

---

## Cached Mode vs Stored Mode

### Cached Mode (Chế Độ Cache — Dữ Liệu Chính Trên S3)

```
Nguyên lý:
┌──────────────────────────────────────────────────────┐
│  Primary Storage: Amazon S3 (toàn bộ dữ liệu)        │
│  Local Cache: Disk on-prem (hot data gần đây)         │
│                                                        │
│  Volume size: Tối đa 32TB/volume, 32 volumes/gateway  │
│  Cache size: Tối đa 1,024 GB per volume               │
│  Total: ~1 PB có thể lưu trên S3                      │
└──────────────────────────────────────────────────────┘

Luồng đọc (cache hit):
App → iSCSI → Gateway → Cache HIT → trả về data (ms)

Luồng đọc (cache miss):
App → iSCSI → Gateway → Cache MISS → S3 fetch → cache → trả về (100-500ms)

Luồng ghi:
App → iSCSI → Gateway → Upload Buffer → xác nhận ghi → async upload S3
```

**Ưu điểm Cached Mode:**
- Dung lượng on-premises nhỏ (chỉ cần cache)
- Total volume size không bị giới hạn bởi local disk
- Phù hợp khi dataset lớn nhưng working set nhỏ

**Nhược điểm Cached Mode:**
- Cache miss → latency cao
- Phụ thuộc vào kết nối Internet/Direct Connect
- Restore toàn bộ volume chậm hơn Stored Mode

### Stored Mode (Chế Độ Lưu Cục Bộ — Dữ Liệu Chính On-Premises)

```
Nguyên lý:
┌──────────────────────────────────────────────────────┐
│  Primary Storage: Local disk on-prem (toàn bộ)       │
│  Async Backup: S3 (EBS Snapshots định kỳ)            │
│                                                        │
│  Volume size: Tối đa 16TB/volume, 32 volumes/gateway  │
│  Upload buffer: Tối thiểu 150 GB                      │
│  Total local: ~512 TB (phụ thuộc hardware)            │
└──────────────────────────────────────────────────────┘

Luồng đọc:
App → iSCSI → Gateway → Local Disk → trả về data (thấp nhất, ms)

Luồng ghi:
App → iSCSI → Gateway → Local Disk (ghi ngay) → Upload Buffer → S3 (async)

Snapshot:
Gateway → EBS Snapshot trong S3 → dùng để create EBS volume khi cần
```

**Ưu điểm Stored Mode:**
- Latency thấp nhất (tất cả reads từ local disk)
- Không phụ thuộc kết nối internet cho normal operations
- Snapshot lên S3 là disaster recovery

**Nhược điểm Stored Mode:**
- Bị giới hạn bởi local disk capacity
- Phải có đủ disk on-premises
- Không scale dễ dàng

### Bảng So Sánh

| Tiêu Chí | Cached Mode | Stored Mode |
|----------|-------------|-------------|
| **Primary storage** | Amazon S3 | Local disk |
| **Latency đọc** | ms (cache hit) / cao (miss) | ms (luôn local) |
| **Latency ghi** | ms (buffer) | ms (local) |
| **Local capacity** | Cache only (~TB) | Full dataset (~TB) |
| **Max volume size** | 32 TB | 16 TB |
| **Offline capability** | Không (cần S3 access) | Có (hoạt động offline) |
| **Use case** | Large dataset, small working set | Low latency, offline tolerance |
| **DR** | S3 là primary → khôi phục nhanh | EBS Snapshot từ S3 |

### Decision Tree (Cây Quyết Định)

```
Tôi cần Volume Gateway mode nào?

Câu 1: Dataset có lớn hơn local disk capacity không?
├── Có → Cached Mode (chỉ cache hot data locally)
└── Không → tiếp câu 2

Câu 2: Ứng dụng có chạy được khi internet chậm/mất không?
├── Phải chạy offline → Stored Mode
└── OK với internet dependency → Cached Mode

Câu 3: Latency read có cực kỳ quan trọng?
├── < 5ms luôn luôn → Stored Mode
└── OK với latency cao khi cache miss → Cached Mode
```

---

## iSCSI

### iSCSI Hoạt Động Như Thế Nào

```
iSCSI = SCSI commands qua TCP/IP network

Thành phần:
├── iSCSI Target: Volume Gateway (server side — phía máy chủ)
├── iSCSI Initiator: Application server (client side — phía máy khách)
└── LUN (Logical Unit Number — Số Hiệu Đơn Vị Logic): Block device

Kết nối:
App Server → TCP port 3260 → Volume Gateway → xuất hiện như /dev/sdb trên Linux

Trên Linux:
# Discover target
iscsiadm -m discovery -t sendtargets -p <gateway-ip>:3260

# Login
iscsiadm -m node --login

# Kiểm tra device
lsblk → thấy /dev/sdb (volume từ gateway)

# Format và mount
mkfs.xfs /dev/sdb
mount /dev/sdb /mnt/iscsi-volume
```

### CHAP Authentication (Xác Thực CHAP)

```
CHAP = Challenge-Handshake Authentication Protocol
(Giao Thức Xác Thực Bằng Thách Thức - Bắt Tay)

Tại sao cần: ngăn unauthorized iSCSI connections

Cấu hình:
1. Trên Volume Gateway console: bật CHAP, đặt secret
2. Trên iSCSI initiator: cấu hình CHAP credentials
3. Mỗi volume có CHAP riêng

Ví dụ initiator config (Linux):
node.session.auth.authmethod = CHAP
node.session.auth.username = myinitiatorname
node.session.auth.password = myinitiatorsecret
```

---

## Snapshots và EBS Integration

### Volume Snapshot (Ảnh Chụp Volume)

```
Snapshot = điểm khôi phục tại một thời điểm
Lưu trên S3 dưới dạng EBS-compatible snapshot

Loại snapshot:
├── Scheduled Snapshot (Snapshot Theo Lịch): tự động theo cron
└── On-demand Snapshot: thủ công khi cần

Hành vi incremental (tăng dần):
├── Snapshot 1: 500 GB (full)
├── Snapshot 2: 2 GB (chỉ phần thay đổi)
├── Snapshot 3: 5 GB (chỉ phần thay đổi)
└── Tiết kiệm storage so với full backup mỗi lần
```

### Restore và Clone

```
3 cách dùng Volume Gateway Snapshot:

1. Restore to same gateway (Khôi Phục Về Same Gateway):
   Snapshot → Volume Gateway → tạo volume mới từ snapshot
   Use case: khôi phục sau lỗi dữ liệu

2. Clone to new Volume Gateway:
   Snapshot S3 → New Volume Gateway → volume mới
   Use case: DR gateway ở datacenter khác

3. Create EBS Volume (Tạo EBS Volume trên AWS):
   Snapshot → EBS Volume → attach EC2
   Use case: migrate workload từ on-prem lên EC2
   → Đây là migration path phổ biến!
```

### Migration Path: On-prem → EC2

```
Bước 1: Cài Volume Gateway, tạo volume iSCSI
Bước 2: App server kết nối iSCSI, ghi data bình thường
Bước 3: Tạo EBS snapshot từ volume
Bước 4: Tạo EBS volume từ snapshot trong AWS
Bước 5: Attach EBS vào EC2, chạy app trên cloud
Bước 6: Decommission (tháo gỡ) on-prem app
```

---

## Use Cases Điển Hình

### 1. Database Backup qua iSCSI

```
Scenario: Oracle Database on-premises
Yêu cầu: Backup hàng đêm không ảnh hưởng production

Giải pháp Stored Mode:
┌────────────────────────────────────────┐
│  Oracle DB Server                       │
│  /dev/sda = Primary (local SAN)        │
│  /dev/sdb = iSCSI → Volume Gateway     │
│                                         │
│  Script nightly:                        │
│  RMAN backup to /dev/sdb               │
│  → Gateway upload lên S3              │
│  → Snapshot EBS hàng đêm              │
└────────────────────────────────────────┘

RTO nếu cần restore:
└── EBS Snapshot → EBS Volume → EC2 Oracle → < 2 giờ
```

### 2. Virtual Machine Backup (Backup Máy Ảo)

```
Scenario: VMware cluster 200 VMs
Yêu cầu: Veeam backup, offsite copy

Trước kia: Veeam backup ra tape, gửi xe tải ra offsite
Sau khi dùng Volume Gateway:
Veeam → iSCSI → Volume Gateway (Stored) → S3 async
→ Tiết kiệm: không cần tape/xe tải/offsite facility
→ RPO: 1 giờ (snapshot hourly)
→ RTO từ 4 ngày → 4 giờ
```

### 3. Cloud Migration Staging

```
Lifecycle:
Phase 1: App chạy on-prem, Volume Gateway lưu data lên S3
Phase 2: Tạo EBS snapshot từ gateway volume
Phase 3: Launch EC2 với EBS từ snapshot → test
Phase 4: Cutover — chuyển hoàn toàn lên AWS
Phase 5: Decommission Volume Gateway

Rủi ro thấp vì:
- Data luôn tồn tại trên cả on-prem và AWS
- Có thể rollback bất kỳ lúc nào trong quá trình
```

---

## Cấu Hình và Triển Khai

### Yêu Cầu Phần Cứng/VM

```
Cached Mode:
├── vCPU: 4 cores minimum, 8 khuyến nghị
├── RAM: 7.5 GB minimum
├── Cache disk: SSD, min 150 GB
├── Upload buffer: SSD, min 150 GB
└── Network: 1 Gbps

Stored Mode:
├── vCPU: 4 cores minimum
├── RAM: 7.5 GB minimum
├── Primary storage: HDD OK (latency local anyway)
├── Upload buffer: SSD, min 150 GB
└── Network: 1 Gbps (upload không ảnh hưởng reads)
```

### Tạo Volume — AWS Console

```
1. Storage Gateway console → Create Gateway
2. Choose: Volume Gateway
3. Choose: Cached / Stored
4. Download OVA (VMware) hoặc image (Hyper-V/KVM)
5. Deploy VM, configure disks
6. Activate gateway với activation key
7. Create volumes, configure iSCSI targets
8. Connect initiators từ app servers
```

---

## Hiệu Suất

### Cached Mode Performance

```
Read performance (IOPS):
├── Cache hit: Limited bởi local disk (SSD = 50,000+ IOPS)
└── Cache miss: Limited bởi S3 download speed

Write performance:
└── Limited bởi upload buffer throughput
   (SSD buffer → ổn định ~100-200 MB/s)

Tối ưu:
├── Dùng SSD cho cả cache và upload buffer
├── Maximize cache size để tăng hit rate
└── Direct Connect để giảm cache miss penalty
```

### Stored Mode Performance

```
Read/Write performance:
└── Tương đương local disk (không phụ thuộc internet)
   SSD primary: 50,000+ IOPS
   HDD primary: 100-200 IOPS (đủ cho backup workload)

Upload to S3:
├── Async — không ảnh hưởng foreground I/O
└── Throttle bởi băng thông internet
```

---

## Bảo Mật

### Encryption

```
Data in-transit (Khi Truyền):
├── iSCSI: CHAP authentication (không mã hóa by default)
│   → Thêm: dùng IPSec hoặc mã hóa tầng ứng dụng
└── Gateway → S3: HTTPS/TLS (tự động)

Data at-rest (Khi Lưu Trữ):
├── S3 objects: SSE-S3 hoặc SSE-KMS
├── Local cache/storage: phụ thuộc disk encryption
└── EBS Snapshots: tùy chọn mã hóa khi tạo
```

### Network Isolation

```
Best practices:
├── Volume Gateway trong security group riêng
├── iSCSI port 3260 chỉ mở cho app servers cụ thể
├── VPC Endpoint cho S3 → không qua internet
└── CloudTrail bật → audit mọi S3 API calls
```

---

## Chi Phí

### Thành Phần Chi Phí

```
Volume Gateway (Cached Mode):
├── Data written to S3: $0.01/GB
├── S3 storage: tùy class
└── EBS Snapshot storage: $0.05/GB-month

Volume Gateway (Stored Mode):
├── Data uploaded to S3: $0.01/GB
├── EBS Snapshot storage: $0.05/GB-month
└── Data transfer in: miễn phí

Ví dụ Stored Mode 10TB database:
- Snapshot 10TB: $0.05 × 10,000 = $500/tháng
- Data write (nightly backup 100GB): $0.01 × 100 = $1/ngày
- So với offsite tape: $2,000+/tháng → tiết kiệm 75%
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Volume Gateway Cached Mode vs Stored Mode khác nhau thế nào?**

> - **Cached Mode**: S3 là primary storage, local chỉ là cache hot data. Dung lượng không giới hạn bởi local disk. Cache miss gây latency cao.
> - **Stored Mode**: Local disk là primary, S3 nhận async backup dưới dạng EBS snapshots. Latency luôn thấp (local), nhưng bị giới hạn bởi local capacity.

**Q2: Khi nào dùng Volume Gateway thay vì File Gateway?**

> Volume Gateway khi ứng dụng cần block storage qua iSCSI — ví dụ database, backup software dùng iSCSI NAS, ứng dụng Windows cần raw block device.
> File Gateway khi ứng dụng dùng NFS/SMB file protocols.

**Q3: Làm sao Volume Gateway hỗ trợ disaster recovery?**

> - **Stored Mode**: Volume Gateway tạo EBS snapshots định kỳ lên S3. Khi DR, tạo EBS volume từ snapshot → attach EC2 → ứng dụng chạy trên cloud.
> - **Cached Mode**: Toàn bộ data trên S3. Khi DR, tạo Volume Gateway mới ở region khác, kết nối với S3 bucket đã có data → recover nhanh hơn.

**Q4: Volume Gateway có thể dùng để migrate app từ on-prem lên EC2 không?**

> Có. Workflow:
> 1. Gài Volume Gateway, app server kết nối iSCSI, chạy bình thường
> 2. Tạo EBS snapshot từ gateway volume
> 3. Restore thành EBS volume
> 4. Attach vào EC2, test ứng dụng
> 5. Cutover khi sẵn sàng

**Q5: iSCSI CHAP là gì và tại sao quan trọng?**

> CHAP (Challenge-Handshake Authentication Protocol) là cơ chế xác thực cho iSCSI connections. Không có CHAP, bất kỳ máy nào trong cùng mạng có thể kết nối iSCSI target và đọc/ghi dữ liệu. CHAP yêu cầu username/password để ngăn unauthorized access — đặc biệt quan trọng trong môi trường shared network.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
