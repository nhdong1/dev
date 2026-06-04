# AWS Storage Gateway — File Gateway, Volume Gateway, Tape Gateway

> **AWS Storage Gateway** là dịch vụ hybrid cloud storage (lưu trữ đám mây lai), cung cấp khả năng tích hợp liền mạch giữa on-premises và AWS Cloud. Ứng dụng on-premises tiếp tục dùng các giao thức quen thuộc (NFS, SMB, iSCSI, VTL) trong khi dữ liệu thực tế được lưu trữ trên AWS — giúp giảm chi phí storage và tăng độ bền dữ liệu mà không cần thay đổi ứng dụng.

## 📚 Mục Lục (Table of Contents)

1. [Storage Gateway là gì và khi nào dùng?](#storage-gateway-là-gì-và-khi-nào-dùng)
2. [Kiến Trúc Chung](#kiến-trúc-chung)
3. [File Gateway — Cổng File NFS/SMB](#file-gateway--cổng-file-nfssmb)
4. [Volume Gateway — Cổng Volume iSCSI](#volume-gateway--cổng-volume-iscsi)
5. [Tape Gateway — Cổng Băng Từ Ảo VTL](#tape-gateway--cổng-băng-từ-ảo-vtl)
6. [Bảng So Sánh Ba Loại Gateway](#bảng-so-sánh-ba-loại-gateway)
7. [Storage Gateway vs DataSync](#storage-gateway-vs-datasync)
8. [Cài Đặt và Triển Khai](#cài-đặt-và-triển-khai)
9. [Monitoring và Troubleshooting](#monitoring-và-troubleshooting)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Storage Gateway là gì và khi nào dùng?

### Storage Gateway là gì?

```
AWS Storage Gateway = Cầu nối hybrid giữa on-premises và AWS

Nguyên lý hoạt động:
├── Cài đặt Storage Gateway appliance (thiết bị/VM) tại on-premises
├── Appliance cung cấp giao thức storage tiêu chuẩn (NFS/SMB/iSCSI/VTL)
├── Ứng dụng on-premises kết nối như storage bình thường
└── Gateway tự động sync/cache dữ liệu lên AWS (S3, EBS, Glacier)

Kết quả từ góc nhìn ứng dụng:
├── Server Linux mount /mnt/data như NFS share bình thường
└── Thực tế: dữ liệu lưu trên S3 Glacier hoặc EBS với local cache
```

### Khi nào dùng Storage Gateway?

```
Phù hợp:
├── Ứng dụng on-premises cần cloud storage nhưng không thể thay đổi code
├── Mở rộng storage on-premises không muốn mua phần cứng mới
├── Backup và DR (Disaster Recovery — Phục Hồi Thảm Họa) từ on-premises
├── Thay thế tape library (thư viện băng từ vật lý) bằng cloud storage
└── Hybrid cloud: một phần dữ liệu on-premises, archive trên AWS

Không phù hợp:
├── Migration một lần → DataSync nhanh hơn
├── SFTP file exchange với đối tác → Transfer Family
└── Ứng dụng đã sẵn sàng dùng S3 API → dùng S3 trực tiếp
```

---

## 🏗️ Kiến Trúc Chung

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     AWS Storage Gateway Architecture                      │
├─────────────────────────────┬────────────────────────────────────────────┤
│      On-Premises            │              AWS Cloud                      │
│                             │                                             │
│  ┌──────────────────────┐   │  ┌─────────────────────────────────────┐  │
│  │  Application Server  │   │  │         AWS Storage Backend          │  │
│  │  (NFS/SMB/iSCSI/VTL) │   │  │                                     │  │
│  └──────────┬───────────┘   │  │  File Gateway   → Amazon S3         │  │
│             │               │  │  Volume Gateway → Amazon S3 + EBS   │  │
│             ▼               │  │  Tape Gateway   → Amazon S3 Glacier │  │
│  ┌──────────────────────┐   │  │                                     │  │
│  │  Storage Gateway     │   │  └─────────────────────────────────────┘  │
│  │  Appliance           │   │              ↑                             │
│  │  (VM hoặc hardware)  │───┼──────────────┘                            │
│  │                      │   │   HTTPS (encrypted transfer)               │
│  │  Local Cache         │   │                                             │
│  │  (hot data tại đây)  │   │                                             │
│  └──────────────────────┘   │                                             │
└─────────────────────────────┴────────────────────────────────────────────┘

Cache layer (lớp cache):
├── Hot data (dữ liệu thường xuyên truy cập) → cache tại local disk
├── Warm/cold data → lấy từ AWS khi cần (cache miss)
└── Gateway tự quản lý cache eviction (đẩy dữ liệu ít dùng khỏi cache)
```

### Deployment Options (Tùy Chọn Triển Khai)

```
Option 1: VMware ESXi
└── Deploy OVA (Open Virtual Appliance) trên ESXi host

Option 2: Microsoft Hyper-V
└── Deploy VHDX image trên Hyper-V

Option 3: Linux KVM (Kernel-based Virtual Machine)
└── Deploy image trên KVM hypervisor

Option 4: Amazon EC2
└── Launch AMI từ AWS Marketplace (dùng cho test hoặc cloud-to-cloud)

Option 5: AWS Storage Gateway Hardware Appliance
└── Thiết bị phần cứng vật lý do AWS cung cấp
└── Dùng khi không có ảo hóa hoặc cần performance vật lý cao
```

---

## 📁 File Gateway — Cổng File NFS/SMB

### File Gateway là gì?

File Gateway (S3 File Gateway) cho phép ứng dụng on-premises mount một network share và lưu file — trong khi thực tế file được lưu dưới dạng S3 objects.

```
File Gateway hoạt động:
├── Gateway cung cấp NFS (Network File System) hoặc SMB (Server Message Block) share
├── Ứng dụng Linux mount qua NFS: mount -t nfs gateway-ip:/share /mnt/data
├── Ứng dụng Windows mount qua SMB: \\gateway-ip\share
├── File write → cached tại local disk → sync lên S3 asynchronously
└── File read:
    ├── Cache hit (có trong local cache): trả về ngay lập tức (fast)
    └── Cache miss (không có cache): tải từ S3 về (latency cao hơn)
```

### File → S3 Object Mapping

```
File path               →  S3 Object key
/mnt/data/finance/      →  s3://company-bucket/finance/
/mnt/data/finance/q1.xlsx → s3://company-bucket/finance/q1.xlsx

Metadata được preserve (giữ lại):
├── Content type (loại nội dung)
├── Last-modified timestamp (dấu thời gian)
└── File permissions (POSIX hoặc Windows ACL)
```

### S3 Storage Classes và Lifecycle

```
File Gateway hỗ trợ chọn S3 storage class cho từng file share:
├── S3 Standard: file thường xuyên truy cập
├── S3 Standard-IA (Infrequent Access — Truy Cập Không Thường Xuyên): backup
└── S3 Intelligent-Tiering: tự động chuyển tier theo access pattern

Kết hợp với S3 Lifecycle Rules (Quy Tắc Vòng Đời S3):
├── 30 ngày không truy cập → chuyển sang S3-IA
└── 90 ngày → chuyển sang S3 Glacier
→ Tiết kiệm chi phí storage đáng kể
```

### Refresh Cache

```
Vấn đề: Nếu file trong S3 bị thay đổi trực tiếp (bypass gateway),
        cache của gateway không biết → đọc sẽ bị outdated

Giải pháp: Refresh cache thủ công hoặc qua API
├── Console: Storage Gateway → File shares → Refresh cache
└── AWS CLI: aws storagegateway refresh-cache --file-share-arn <arn>

Best practice: Nếu nhiều gateway cùng mount một S3 bucket →
              chỉ nên một gateway có write permission, các gateway khác read-only
```

### Use Cases File Gateway

```
Use case 1: File server backup lên S3
├── File server on-premises ghi file qua NFS → S3
└── Cost: giảm so với mua thêm NAS hardware

Use case 2: Data lake ingestion (nhập dữ liệu vào hồ dữ liệu)
├── Edge servers ghi dữ liệu qua NFS → S3 tự động
└── Spark/Athena phân tích trực tiếp từ S3

Use case 3: Shared storage cho EC2 và on-premises
├── EC2 instances truy cập S3 qua S3 API
└── On-premises servers truy cập cùng data qua File Gateway NFS
```

---

## 💿 Volume Gateway — Cổng Volume iSCSI

### Volume Gateway là gì?

Volume Gateway cung cấp iSCSI (Internet Small Computer Systems Interface — Giao Thức Truy Cập Block Storage Qua TCP/IP) block storage cho ứng dụng on-premises — tương tự như SAN (Storage Area Network).

```
Volume Gateway hoạt động:
├── Gateway tạo iSCSI targets (thiết bị lưu trữ ảo)
├── Server on-premises kết nối qua iSCSI initiator (Windows/Linux built-in)
├── OS nhận ra như local disk — có thể format, partition bình thường
└── Dữ liệu được lưu trên AWS (S3 hoặc EBS Snapshots) tùy chế độ
```

### Hai Chế Độ Volume Gateway

#### Stored Volumes (Volume Lưu Tại Chỗ)

```
Stored Volumes:
├── Dữ liệu CHÍNH lưu tại on-premises (local disk của gateway VM)
├── AWS chỉ lưu backup dưới dạng EBS Snapshots (ảnh chụp ổ đĩa)
├── Low latency (độ trễ thấp) cho tất cả reads/writes → phù hợp I/O cao
└── Use case: primary storage on-premises + backup lên AWS

Đặc điểm:
├── Volume size: 1 GB - 16 TB per volume
├── Tối đa 32 volumes per gateway
└── Snapshot incremental: chỉ backup phần thay đổi (tiết kiệm chi phí)

Timeline:
Local disk ← Read/Write (nhanh, không qua mạng)
     │
     └── Async backup → EBS Snapshot (S3 under the hood)
         (Backup định kỳ hoặc triggered manually)
```

#### Cached Volumes (Volume Ưu Tiên Cache)

```
Cached Volumes:
├── Dữ liệu CHÍNH lưu trên S3 (AWS là primary storage)
├── On-premises chỉ cache hot data (dữ liệu thường xuyên dùng)
├── Cache miss → tải từ S3 (có latency từ mạng)
└── Use case: mở rộng storage mà không mua thêm phần cứng

Đặc điểm:
├── Volume size: 1 GB - 32 TB per volume
├── Cache size: tùy disk local gateway VM
└── Phù hợp khi dữ liệu lớn nhưng chỉ access một phần nhỏ thường xuyên

Timeline:
Local cache ← Read hot data (nhanh)
S3 ← Read cold data (qua mạng) + Write (async)
```

### So sánh Stored vs Cached

| Tiêu Chí | Stored Volumes | Cached Volumes |
| -------- | -------------- | -------------- |
| **Primary storage** | On-premises local disk | AWS S3 |
| **AWS dùng cho** | EBS Snapshots (backup) | Primary storage |
| **Read latency** | Luôn thấp (local disk) | Thấp với hot data, cao với cold data |
| **Capacity giới hạn bởi** | Local disk size | S3 (không giới hạn) |
| **Chi phí local hardware** | Cao (phải có disk lớn) | Thấp (chỉ cần cache disk) |
| **Use case** | Low-latency I/O, primary local | Scale storage, ít access random |

### Volume Recovery

```
Khi on-premises bị sự cố (disaster):
├── Stored Volumes: restore từ EBS Snapshot → tạo EBS volume mới → attach EC2
└── Cached Volumes: restore từ S3 → tạo volume mới (full data có trên S3)

Cross-region DR:
├── Copy EBS Snapshot sang region khác
└── Restore tại region mới → tiếp tục hoạt động
```

---

## 📼 Tape Gateway — Cổng Băng Từ Ảo VTL

### Tape Gateway là gì?

Tape Gateway (VTL — Virtual Tape Library — Thư Viện Băng Từ Ảo) giả lập thư viện băng từ vật lý (physical tape library) tương thích với các phần mềm backup phổ biến.

```
Tape Gateway hoạt động:
├── Gateway giả lập thiết bị tape: VTL (chứa nhiều VT — Virtual Tapes)
│   và VTS (Virtual Tape Shelf — Kệ Băng Từ Ảo)
├── Phần mềm backup (Veeam, Commvault, NetBackup, Backup Exec)
│   "nghĩ" đang ghi vào tape thật → không cần thay đổi backup policy
├── Dữ liệu thực tế: ghi vào S3 (active tapes) → archive sang S3 Glacier
└── Không còn cần mua băng từ vật lý, máy đọc băng, bảo quản vật lý
```

### Virtual Tape (Băng Từ Ảo) Lifecycle

```
Tape lifecycle:
1. Tạo Virtual Tape (VT):
   ├── Size: 100 GB - 5 TB per tape
   └── Lưu trên S3 (trong VTL)

2. Backup job viết vào tape:
   └── Data ghi vào local buffer → sync lên S3

3. Archive tape (lưu trữ dài hạn):
   ├── Eject tape từ VTL vào VTS (Virtual Tape Shelf)
   └── AWS di chuyển từ S3 sang S3 Glacier Flexible Retrieval
       hoặc S3 Glacier Deep Archive (rẻ nhất, lấy lại sau 12h-48h)

4. Retrieve tape (lấy lại để restore):
   ├── Request tape từ VTS về VTL (mất vài giờ với Glacier)
   └── Phần mềm backup thực hiện restore từ tape

5. Delete tape:
   └── Xóa khi không còn cần

Pricing insight:
├── S3 (active tape): $0.023/GB/tháng
└── S3 Glacier Deep Archive (archived tape): $0.00099/GB/tháng
    → Tiết kiệm ~23× so với active S3
```

### Phần Mềm Backup Tương Thích

```
Tape Gateway tương thích với phần mềm:
├── Veeam Backup & Replication — phổ biến nhất trong VMware environment
├── Commvault Complete Backup & Recovery
├── Veritas NetBackup — enterprise scale
├── Veritas Backup Exec — SMB environment
├── IBM Spectrum Protect (TSM)
├── Dell Technologies (EMC) NetWorker
└── Microsoft System Center Data Protection Manager (DPM)

Không cần thay đổi cấu hình backup:
└── Chỉ trỏ phần mềm backup đến gateway IP thay vì physical tape device
```

### Use Case điển hình

```
Bài toán: Công ty có 50 TB backup tape vật lý, muốn migrate lên AWS:

Bước 1: Cài Tape Gateway (VM on-premises)
Bước 2: Cấu hình phần mềm backup → dùng VTL thay vì physical tape
Bước 3: Backup mới ghi vào VTL → S3 tự động
Bước 4: Dữ liệu cũ trên physical tape → copy lên S3 Glacier thủ công (hoặc DataSync)
Bước 5: Physical tape library → có thể retire (loại bỏ)

Tiết kiệm:
├── Không mua băng mới (~$20-50/băng vật lý)
├── Không thuê offsite storage (lưu trữ băng ngoài site)
├── Không bảo trì máy đọc băng
└── S3 Glacier Deep Archive: ~$0.99/TB/tháng (vs $5-15/TB/tháng offsite tape)
```

---

## 📊 Bảng So Sánh Ba Loại Gateway

| Tiêu Chí | File Gateway | Volume Gateway | Tape Gateway |
| -------- | ------------ | -------------- | ------------ |
| **Giao thức** | NFS, SMB | iSCSI | iSCSI VTL |
| **Storage unit** | File (object) | Block (volume) | Tape (virtual) |
| **AWS backend** | S3 | S3 + EBS Snapshots | S3 + S3 Glacier |
| **Cache tại local** | ✅ Hot files | ✅ Hot blocks | ✅ Active tape data |
| **Use case** | File server, NAS, data lake | Block storage, DB backup | Replace physical tape library |
| **Backup software** | Không cần | Không cần | ✅ Cần (Veeam, Commvault...) |
| **Access pattern** | File-level | Block-level | Sequential (tape) |

---

## ⚖️ Storage Gateway vs DataSync

Câu hỏi hay gặp trong phỏng vấn: Storage Gateway khác DataSync thế nào?

```
Điểm khác biệt cốt lõi:

DataSync:
├── Mục đích: Di chuyển/đồng bộ dữ liệu theo lịch (scheduled/one-time)
├── Hướng: Thường một chiều (on-premises → AWS hoặc AWS → AWS)
├── Tốc độ: Optimized cho throughput cao (bulk transfer)
└── Dùng cho: Migration, backup định kỳ, archiving

Storage Gateway:
├── Mục đích: Tích hợp hybrid liên tục — ứng dụng dùng cloud storage như local
├── Hướng: Hai chiều, real-time (write → AWS, read ← cache hoặc AWS)
├── Tốc độ: Ưu tiên low latency cho hot data (nhờ local cache)
└── Dùng cho: Extend storage, replace tape, hybrid cloud persistent integration

Quy tắc chọn:
├── Cần "di chuyển" dữ liệu → DataSync
└── Ứng dụng cần "dùng" cloud storage như local liên tục → Storage Gateway
```

---

## 🛠️ Cài Đặt và Triển Khai

### Quy trình cài đặt chung

```
Bước 1: Download gateway image từ AWS Console
├── Chọn loại gateway: File / Volume / Tape
└── Chọn hypervisor: VMware / Hyper-V / KVM

Bước 2: Deploy VM trên hypervisor
├── Cấu hình yêu cầu tối thiểu:
│   ├── vCPU: 4
│   ├── RAM: 16 GB (khuyến nghị 32 GB cho production)
│   ├── Root disk: 80 GB (OS + software)
│   └── Cache disk: Tùy theo nhu cầu (ví dụ: 150 GB - 16 TB)
└── Network: 1 interface có thể ra internet (port 443)

Bước 3: Activate (kích hoạt) gateway
├── Mở browser, trỏ đến IP của VM → cổng 80 (chỉ dùng lần kích hoạt)
├── AWS Console → Storage Gateway → Get started → Activate gateway
└── Gateway xuất hiện trong console với trạng thái "Running"

Bước 4: Cấu hình cache và upload buffer
├── Thêm disk vào VM
├── Gán disk làm "Cache" hoặc "Upload buffer"
└── Upload buffer: lưu tạm dữ liệu trước khi upload AWS (thường 150 GB)

Bước 5: Tạo file share / volume / tape
└── Tùy loại gateway, tạo NFS share, iSCSI target, hoặc VTL
```

### Networking Requirements (Yêu Cầu Mạng)

```
Gateway cần outbound HTTPS (port 443) đến:
├── AWS Storage Gateway service endpoints
├── S3 endpoints
└── CloudWatch endpoints

Khuyến nghị:
├── AWS Direct Connect hoặc VPN cho môi trường production
└── VPC Endpoints để traffic không ra internet
```

---

## 📊 Monitoring và Troubleshooting

### CloudWatch Metrics quan trọng

```
File Gateway metrics:
├── CacheHitPercent: % requests phục vụ từ cache (mục tiêu > 90%)
├── CachePercentUsed: % cache đang dùng (cảnh báo khi > 80%)
└── ReadBytes / WriteBytes: throughput thực tế

Volume Gateway metrics:
├── CacheHitPercent: tương tự File Gateway
├── ReadTime / WriteTime: latency đọc/ghi
└── QueuedWriteBytes: dữ liệu đang chờ upload lên AWS

Tape Gateway metrics:
└── TapeUsed: dung lượng tapes đang dùng
```

### Vấn đề phổ biến

```
1. Low cache hit rate (Tỷ lệ cache hit thấp):
   ├── Nguyên nhân: cache disk quá nhỏ so với working set (tập dữ liệu đang dùng)
   └── Giải pháp: thêm disk lớn hơn cho cache

2. High write latency (Độ trễ ghi cao):
   ├── Nguyên nhân: mạng chậm đến AWS hoặc upload buffer đầy
   └── Giải pháp: Direct Connect / nâng băng thông / tăng upload buffer

3. Gateway offline:
   ├── Nguyên nhân: VM crash, network block port 443
   └── Giải pháp: kiểm tra VM, security group, DNS resolution

4. File không sync lên S3:
   └── Kiểm tra IAM role của gateway có quyền ghi S3 không
```

---

## 🎓 Câu Hỏi Phỏng Vấn

1. **Phân biệt ba loại Storage Gateway. Khi nào dùng từng loại?**
   - File Gateway: ứng dụng cần NFS/SMB share, file lưu trên S3. Volume Gateway: block storage iSCSI, backup dưới dạng EBS snapshots. Tape Gateway: thay thế physical tape library, tương thích Veeam/Commvault.

2. **File Gateway khác DataSync thế nào?**
   - File Gateway: tích hợp hybrid liên tục, ứng dụng dùng như local storage (NFS/SMB), có local cache. DataSync: bulk transfer/đồng bộ theo lịch, tối ưu throughput, không có persistent integration.

3. **Volume Gateway có hai chế độ. Phân biệt Stored vs Cached?**
   - Stored: primary storage on-premises (local disk), AWS chỉ lưu backup EBS snapshots → latency thấp luôn. Cached: primary storage trên S3 (AWS), on-premises chỉ cache hot data → scale không giới hạn nhưng cold data có latency cao hơn.

4. **Tape Gateway lưu dữ liệu ở đâu? Tại sao tiết kiệm chi phí?**
   - Active tape: S3. Archived tape: S3 Glacier Deep Archive ($0.00099/GB/tháng). Không cần mua băng vật lý, máy đọc băng, offsite storage. Tiết kiệm hàng chục lần so với tape vật lý.

5. **Ứng dụng on-premises ghi file qua File Gateway, sau đó team data engineer muốn đọc từ S3. Có vấn đề gì không?**
   - Hoạt động bình thường — file trên File Gateway được lưu dưới dạng S3 object, team data engineer đọc qua S3 API. Không cần thay đổi gì.

6. **Khi cache đầy trong File Gateway, điều gì xảy ra?**
   - Gateway tự động evict (loại bỏ) cold data (dữ liệu ít truy cập) khỏi cache để nhường chỗ cho hot data mới. Dữ liệu bị evict không mất — vẫn còn trên S3. Lần đọc tiếp theo sẽ tải lại từ S3 (cache miss với latency cao hơn).

7. **Giải thích upload buffer trong Storage Gateway là gì.**
   - Upload buffer là vùng disk tạm thời trên gateway VM, lưu trữ dữ liệu vừa ghi trước khi được upload lên AWS. Cần đủ lớn để không overflow trong trường hợp mạng chậm tạm thời. Khuyến nghị tối thiểu 150 GB.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
