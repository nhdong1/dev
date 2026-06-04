# So Sánh Toàn Diện FSx — Chọn Đúng Biến Thể

> Tài liệu này so sánh chi tiết bốn biến thể FSx (Windows, Lustre, NetApp ONTAP, OpenZFS), giúp chọn đúng dịch vụ cho từng use case, kèm decision tree và ví dụ phỏng vấn thực tế.

---

## Mục Lục

1. [Bảng So Sánh Tổng Hợp](#bảng-so-sánh-tổng-hợp)
2. [Decision Tree — Cây Quyết Định](#decision-tree---cây-quyết-định)
3. [So Sánh Theo Từng Tiêu Chí](#so-sánh-theo-từng-tiêu-chí)
4. [FSx vs EFS vs EBS — Khi Nào Dùng Gì](#fsx-vs-efs-vs-ebs---khi-nào-dùng-gì)
5. [Scenarios Phỏng Vấn](#scenarios-phỏng-vấn)
6. [Checklist Chọn FSx](#checklist-chọn-fsx)

---

## Bảng So Sánh Tổng Hợp

| Tiêu Chí | FSx Windows | FSx Lustre | FSx NetApp ONTAP | FSx OpenZFS |
|---------|-------------|------------|-----------------|-------------|
| **Protocol chính** | SMB | Lustre | NFS + SMB + iSCSI | NFS |
| **OS hỗ trợ** | Windows & Linux | Linux only | Windows & Linux | Linux & macOS |
| **Multi-protocol** | Không | Không | ✅ Có | Không |
| **Active Directory** | ✅ Native | Không | ✅ Có | Không |
| **Deployment** | Single/Multi-AZ | Single-AZ | Multi-AZ only | Single-AZ |
| **Throughput tối đa** | 2 GB/s | Hàng trăm GB/s | 4 GB/s | 21 GB/s |
| **IOPS tối đa** | ~1M | Hàng triệu | Hàng triệu | Hàng triệu |
| **Latency (Độ trễ)** | < 1ms | Sub-ms | Sub-ms | Sub-ms |
| **Storage tối đa** | 64 TB | Petabytes | Petabytes | 64 TB |
| **Snapshot** | VSS (manual) | Không | SnapMirror | Instant (CoW) |
| **Clone** | Không | Không | FlexClone | ✅ Có |
| **Deduplication** | ✅ Có | Không | ✅ Có | ✅ Có |
| **Compression** | ✅ Có | Không | ✅ Có | ✅ LZ4/ZSTD |
| **S3 Integration** | Không | ✅ Native | Không | Không |
| **On-prem Migration** | Windows FS | Không | ✅ SnapMirror | ✅ ZFS send |
| **Giá storage (SSD)** | ~$0.13/GB | $0.14-0.36/GB | ~$0.125/GB | ~$0.09/GB |
| **Phù hợp nhất** | Windows WL | HPC / ML | Enterprise | Dev/Test |

---

## Decision Tree — Cây Quyết Định

```
Cần FSx? Bắt đầu từ đây:

Workload yêu cầu Windows / SMB / Active Directory?
├── Có → FSx for Windows File Server
└── Không ↓

Workload HPC / ML Training / Genomics / Rendering?
├── Có → FSx for Lustre
│         ├── Dữ liệu tạm thời → Scratch deployment
│         └── Dữ liệu lâu dài → Persistent deployment
└── Không ↓

Đang dùng NetApp on-premises, cần di chuyển lên cloud?
├── Có → FSx for NetApp ONTAP
└── Không ↓

Cần multi-protocol (NFS + SMB + iSCSI cùng lúc)?
├── Có → FSx for NetApp ONTAP
└── Không ↓

Cần snapshot instant, cloning cho dev/test?
├── Có → FSx for OpenZFS
└── Không ↓

Workload Linux thuần túy, shared NFS, không cần tính năng đặc biệt?
└── EFS (không cần FSx)
```

---

## So Sánh Theo Từng Tiêu Chí

### 1. Protocol và OS Support (Giao Thức và Hỗ Trợ Hệ Điều Hành)

```
SMB — Server Message Block (Windows share):
├── FSx Windows ✅ (native)
├── FSx ONTAP ✅
├── FSx Lustre ❌
└── FSx OpenZFS ❌

NFS — Network File System (Linux/Unix):
├── FSx Windows ✅ (qua NFS client, hạn chế)
├── FSx ONTAP ✅ (native, v3 + v4.1)
├── FSx Lustre ❌ (dùng Lustre protocol)
└── FSx OpenZFS ✅ (native, v3 + v4.2)

iSCSI — Block storage qua TCP/IP:
├── FSx Windows ❌
├── FSx ONTAP ✅
├── FSx Lustre ❌
└── FSx OpenZFS ❌

Lustre client (HPC parallel):
├── FSx Windows ❌
├── FSx ONTAP ❌
├── FSx Lustre ✅ (native)
└── FSx OpenZFS ❌
```

### 2. High Availability — Tính Sẵn Sàng Cao

```
Multi-AZ tự động:
├── FSx Windows  → Có option Single-AZ và Multi-AZ
│                  Multi-AZ: Failover tự động < 30s
├── FSx Lustre   → Single-AZ only (không có Multi-AZ)
├── FSx ONTAP    → Multi-AZ only (HA Pair bắt buộc)
└── FSx OpenZFS  → Single-AZ only (không có Multi-AZ)

Khuyến nghị cho Production Critical:
→ FSx Windows (Multi-AZ) hoặc FSx ONTAP
→ FSx Lustre và OpenZFS: không phù hợp nếu AZ failure là risk
```

### 3. Hiệu Suất (Performance)

```
Throughput cao nhất:
#1 FSx Lustre: Hàng trăm GB/s (không giới hạn khi scale)
#2 FSx OpenZFS: 21 GB/s (đọc), 1.5 GB/s (ghi)
#3 FSx ONTAP: 4 GB/s
#4 FSx Windows: 2 GB/s

IOPS cao nhất:
Tất cả đều > 1 triệu IOPS ở cấu hình cao
→ FSx Lustre tốt nhất cho parallel I/O với nhiều clients

Latency:
Tất cả đều sub-millisecond cho SSD storage
→ FSx Lustre: < 0.1ms (microsecond range cho some operations)
```

### 4. Storage Efficiency (Hiệu Quả Lưu Trữ)

```
Deduplication + Compression:
┌───────────────┬──────────┬─────────────┬───────────────────────────┐
│ Biến Thể      │ Dedup    │ Compression │ Hiệu Quả Điển Hình       │
├───────────────┼──────────┼─────────────┼───────────────────────────┤
│ FSx Windows   │ ✅ Inline │ ✅ Inline   │ 2:1 đến 5:1               │
│ FSx Lustre    │ ❌        │ ❌          │ 1:1 (không nén)           │
│ FSx ONTAP     │ ✅ Inline │ ✅ Adaptive │ 2:1 đến 10:1              │
│ FSx OpenZFS   │ ✅ (cần RAM│ ✅ LZ4/ZSTD│ 1.5:1 đến 4:1            │
└───────────────┴──────────┴─────────────┴───────────────────────────┘

Lưu ý: FSx Lustre không có storage efficiency features
→ Design để tối đa throughput, không phải storage density
```

### 5. Chi Phí (Pricing)

```
Giá lưu trữ (approximate, SSD):
├── OpenZFS:  $0.09/GB-month  (rẻ nhất)
├── ONTAP:    $0.125/GB-month
├── Windows:  $0.13/GB-month
└── Lustre:   $0.14-0.36/GB-month (tùy deployment type)

Tổng cost ownership khi có storage efficiency:
Nếu ONTAP đạt dedup ratio 3:1:
→ Chi phí thực tế: $0.125 / 3 = $0.042/GB-logical
→ Rẻ hơn OpenZFS cho data có nhiều duplicate

Lưu ý về Lustre:
→ Thường được spin up ngắn hạn (giờ đến ngày), không tháng dài
→ $0.14/GB/month → $0.19/GB/30 days → $0.0063/GB/day
→ Training 5 TB trong 3 ngày: 5.000 × $0.0063 × 3 = $94 tổng
```

---

## FSx vs EFS vs EBS — Khi Nào Dùng Gì

### Bảng Quyết Định Cuối

| Yêu Cầu | Giải Pháp |
|---------|-----------|
| Windows workload + AD | FSx Windows |
| HPC / ML / GPU training | FSx Lustre |
| Migration từ NetApp | FSx ONTAP |
| Multi-protocol NFS+SMB+iSCSI | FSx ONTAP |
| Dev/test + snapshot/clone | FSx OpenZFS |
| Linux shared storage, đơn giản | EFS |
| Linux shared storage, tiết kiệm chi phí, POSIX đầy đủ | EFS |
| Database trên EC2, low latency | EBS |
| Boot volume EC2 | EBS |
| Static files, backup, data lake | S3 |

### Ma Trận Đầy Đủ

```
                    │ S3  │ EBS │ EFS │ FSx-W │ FSx-L │ FSx-O │ FSx-Z │
────────────────────┼─────┼─────┼─────┼───────┼───────┼───────┼───────┤
Object storage      │  ✅  │  ❌  │  ❌  │   ❌   │   ❌   │   ❌   │   ❌   │
Block storage       │  ❌  │  ✅  │  ❌  │   ❌   │   ❌   │   ✅*  │   ❌   │
File storage (NFS)  │  ❌  │  ❌  │  ✅  │   △   │   ❌   │   ✅   │   ✅   │
File storage (SMB)  │  ❌  │  ❌  │  ❌  │   ✅   │   ❌   │   ✅   │   ❌   │
Low latency DB      │  ❌  │  ✅  │  △  │   △   │   ✅   │   ✅   │   ✅   │
Shared multi-EC2    │  ✅  │  △* │  ✅  │   ✅   │   ✅   │   ✅   │   ✅   │
Active Directory    │  ❌  │  ❌  │  ❌  │   ✅   │   ❌   │   ✅   │   ❌   │
HPC / ML training   │  ❌  │  ❌  │  △  │   ❌   │   ✅   │   ❌   │   ❌   │
Snapshot instant    │  ✅  │  ✅  │  ❌  │   △   │   ❌   │   ❌   │   ✅   │
ZFS clones          │  ❌  │  ❌  │  ❌  │   ❌   │   ❌   │   ❌   │   ✅   │
Tiering to S3       │  ✅  │  ❌  │  ✅  │   ❌   │   ✅   │   ✅   │   ❌   │

* EBS Multi-Attach chỉ io1/io2
* FSx ONTAP có iSCSI (block-like)
✅ = Tốt   △ = Có nhưng không tối ưu   ❌ = Không hỗ trợ

FSx-W=Windows, FSx-L=Lustre, FSx-O=ONTAP, FSx-Z=OpenZFS
```

---

## Scenarios Phỏng Vấn

### Scenario 1: Di Chuyển Windows File Server On-Premises

```
Đề bài:
"Công ty bạn có 50 Windows file servers on-premises,
mỗi server ~5 TB SMB shares, tất cả join domain corp.example.com.
Làm sao migrate lên AWS?"

Phân tích:
├── Windows? SMB? Active Directory? → FSx Windows
├── 50 servers × 5 TB = 250 TB → nhiều FSx instances hoặc một instance lớn
└── Không downtime? → DFS Namespace để chuyển hướng sau migration

Giải pháp:
1. AWS Managed Microsoft AD (join vào corp.example.com trust)
2. FSx Windows Multi-AZ, đủ capacity
3. AWS DataSync để di chuyển dữ liệu (không downtime)
4. DFS Namespace update để chuyển hướng \\corp\shares → FSx
5. Validate → Decommission on-premises servers
```

### Scenario 2: ML Training Infrastructure

```
Đề bài:
"Team Data Science cần train model trên 10 TB dữ liệu ảnh trong S3.
Sử dụng cluster 32 GPU servers. Training mất 2 ngày.
Sau đó không cần storage nữa."

Phân tích:
├── HPC / ML → FSx Lustre
├── Dữ liệu từ S3 → S3 integration
├── Chỉ dùng 2 ngày → Scratch (không cần persistent)
└── 32 GPU servers cần parallel IO → Lustre phù hợp

Giải pháp:
1. Tạo FSx Lustre Scratch 2, 12 TB (để đủ cho training)
2. Import từ S3: ImportPath = s3://training-data/images/
3. Mount trên tất cả 32 GPU servers
4. Training script đọc từ /mnt/fsx/images/ (lazy loading từ S3)
5. Sau training, export model về S3
6. Xóa FSx Lustre

Chi phí: 12.000 GB × $0.14 × (2/30) = $112 (so với EC2 Instance Store: cần provision 10 TB trên mỗi instance)
```

### Scenario 3: Enterprise Database Migration

```
Đề bài:
"Oracle database 15 TB chạy trên NetApp All-Flash FAS on-premises.
Đang dùng NFS v4 + Oracle ASM (Automatic Storage Management).
Migration lên AWS với downtime tối đa 30 phút."

Phân tích:
├── NetApp on-premises → FSx ONTAP (cùng platform)
├── NFS v4 → FSx ONTAP hỗ trợ
├── Oracle → cần NFS với POSIX đầy đủ và locking
└── < 30 phút downtime → SnapMirror sync trước, cutover nhanh

Giải pháp:
1. Tạo FSx ONTAP, cấu hình SVM giống on-premises
2. Thiết lập SnapMirror relationship (async)
3. Initial sync: 15 TB → vài giờ (qua Direct Connect 10Gbps)
4. Chạy SnapMirror liên tục để sync delta
5. Maintenance window:
   a. Dừng Oracle (5 phút)
   b. Final SnapMirror sync (< 1 phút nếu lag thấp)
   c. Break SnapMirror
   d. Cập nhật /etc/fstab trên EC2 → mount FSx ONTAP
   e. Khởi động Oracle (5 phút)
   → Tổng: < 15 phút downtime
```

### Scenario 4: Dev/Test Database Environment

```
Đề bài:
"Team 20 developer cần môi trường dev với production data (2 TB PostgreSQL).
Mỗi ngày refresh lại. Không được ảnh hưởng production.
Ngân sách hạn chế."

Phân tích:
├── Snapshot + Clone → FSx OpenZFS
├── 20 developer × 2 TB = 40 TB nếu copy → quá đắt
├── Clone: 20 clones × ~0 GB ban đầu → rẻ hơn nhiều
└── Daily refresh → scheduled snapshot + re-clone

Giải pháp:
1. Production DB chạy trên FSx OpenZFS
2. Cron job 2 giờ sáng: tạo snapshot "daily-refresh"
3. Script xóa clones cũ và tạo 20 clones mới từ snapshot
4. Developer mount clone của mình: /fsx/clones/dev-username
5. Mỗi developer có 2 TB data "fresh" mỗi sáng

Chi phí thêm (so với không có dev env):
→ 20 clones × 100 GB changes/ngày = 2.000 GB extra = $180/month
→ So với 20 bản copy: 20 × 2.000 GB = $3.600/month
→ Tiết kiệm $3.420/month
```

### Scenario 5: Mixed Workload Enterprise

```
Đề bài:
"Công ty cần:
- File shares cho Windows users (1.000 nhân viên)
- Shared storage cho Linux containers (Kubernetes)
- iSCSI storage cho legacy application
- Không muốn quản lý nhiều storage systems"

Phân tích:
├── Windows + Linux + iSCSI → multi-protocol → FSx ONTAP
├── Một file system, nhiều SVM, nhiều protocol
└── Single management plane

Giải pháp:
FSx ONTAP với 3 SVM:
├── SVM-Windows: SMB shares cho 1.000 nhân viên + AD
├── SVM-Linux: NFS v4 cho Kubernetes PVC
└── SVM-Legacy: iSCSI LUNs cho legacy app

→ Tất cả trên một FSx ONTAP file system
→ Quản lý tập trung qua AWS Console
→ Backup tự động cho tất cả
```

---

## Checklist Chọn FSx

### Câu Hỏi Để Xác Định Đúng FSx

```
1. Hệ điều hành của workload?
   □ Windows only → FSx Windows
   □ Linux only → Lustre, ONTAP, hoặc OpenZFS
   □ Mixed Windows + Linux → FSx ONTAP

2. Giao thức cần thiết?
   □ SMB → FSx Windows hoặc FSx ONTAP
   □ NFS → FSx ONTAP, OpenZFS, hoặc EFS
   □ iSCSI → FSx ONTAP only
   □ Lustre → FSx Lustre only
   □ Nhiều giao thức cùng lúc → FSx ONTAP only

3. Tính năng đặc thù?
   □ Active Directory native → FSx Windows
   □ S3 integration → FSx Lustre
   □ Snapshot instant + Clone → FSx OpenZFS
   □ SnapMirror, FlexClone, dedup → FSx ONTAP
   □ Throughput hàng trăm GB/s → FSx Lustre

4. High Availability yêu cầu?
   □ Multi-AZ bắt buộc → FSx Windows (Multi-AZ) hoặc FSx ONTAP
   □ Single-AZ ok → Tất cả các loại

5. Nguồn gốc workload?
   □ Đang dùng Windows File Server → FSx Windows
   □ Đang dùng NetApp ONTAP → FSx ONTAP
   □ Đang dùng ZFS/OpenZFS → FSx OpenZFS
   □ Workload mới HPC/ML → FSx Lustre

6. Thời gian sử dụng storage?
   □ Ngắn hạn (giờ đến ngày) → FSx Lustre Scratch
   □ Dài hạn → FSx Lustre Persistent, Windows, ONTAP, OpenZFS
```

---

## Tóm Tắt Một Câu Cho Từng Biến Thể

| FSx | Một Câu Tóm Tắt |
|-----|----------------|
| **FSx Windows** | "NetApp không, Windows có: SMB + Active Directory cho ứng dụng Windows." |
| **FSx Lustre** | "Hàng trăm GB/s với S3 backend cho HPC và ML training." |
| **FSx ONTAP** | "NetApp on-premises chuyển lên cloud, mang theo tất cả tính năng doanh nghiệp." |
| **FSx OpenZFS** | "Snapshot tức thì và clones rẻ tiền cho dev/test environments." |

---

## Điểm Kiểm Tra Phỏng Vấn Tổng Hợp

### Top 5 Câu Hỏi Về FSx

**Q: Giải thích sự khác biệt giữa 4 biến thể FSx.**
> A: (Dùng bảng so sánh trên) — Tập trung vào protocol, use case, và tính năng đặc thù. Windows cho SMB+AD, Lustre cho HPC+S3, ONTAP cho multi-protocol+enterprise, OpenZFS cho snapshot+clone.

**Q: Khi nào chọn FSx thay vì EFS?**
> A: EFS khi cần shared NFS đơn giản cho Linux, serverless, trả theo dùng. FSx khi cần: Windows workload (FSx Windows), HPC/ML (FSx Lustre), multi-protocol (FSx ONTAP), snapshot/clone (FSx OpenZFS), hoặc tính năng enterprise mà EFS không có.

**Q: FSx ONTAP và FSx Windows cùng hỗ trợ SMB — nên chọn cái nào?**
> A: FSx Windows khi workload 100% Windows, cần deep AD integration, SMB shares đơn giản. FSx ONTAP khi cần thêm NFS hoặc iSCSI cùng lúc, cần NetApp features (dedup, FlexClone, SnapMirror), hoặc migration từ NetApp.

**Q: Chi phí FSx so với tự quản lý trên EC2?**
> A: FSx managed service bao gồm HA, backup, patching — tính tổng total cost of ownership (TCO — Tổng Chi Phí Sở Hữu) bao gồm cả nhân lực thì thường ngang hoặc rẻ hơn. Chỉ xét tiền hạ tầng thì tự quản lý thường rẻ hơn, nhưng tốn thêm engineering time.

**Q: Làm sao chọn FSx Lustre Scratch vs Persistent?**
> A: Scratch khi xử lý dữ liệu tạm thời trong vài giờ đến ngày, sau đó không cần nữa — rẻ nhất và throughput cao nhất. Persistent khi training data dùng lặp lại nhiều lần bởi nhiều cluster, hoặc cần data survive khi cluster bị xóa — tốn kém hơn nhưng dữ liệu được nhân bản.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Trạng Thái:** ✅ Hoàn thành
