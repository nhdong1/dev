# 📁 File Gateway — Cổng Lưu Trữ Tệp NFS/SMB → S3

> File Gateway (Cổng Lưu Trữ Tệp) là loại AWS Storage Gateway cho phép ứng dụng on-premises truy cập Amazon S3 thông qua giao thức NFS (Network File System — Hệ Thống Tệp Mạng) hoặc SMB (Server Message Block — Giao Thức Chia Sẻ Tệp Windows) quen thuộc, mà không cần thay đổi code ứng dụng.

---

## 📚 Mục Lục

1. [Kiến Trúc File Gateway](#kiến-trúc-file-gateway)
2. [NFS vs SMB — Chọn Giao Thức Nào](#nfs-vs-smb)
3. [Cache Cục Bộ — Local Cache](#cache-cục-bộ)
4. [Cấu Hình File Share](#cấu-hình-file-share)
5. [Use Cases Điển Hình](#use-cases-điển-hình)
6. [Hiệu Suất và Tối Ưu](#hiệu-suất-và-tối-ưu)
7. [Bảo Mật](#bảo-mật)
8. [Chi Phí](#chi-phí)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc File Gateway

### Luồng Dữ Liệu

```
Ứng Dụng On-Premises
        │
        │  NFS v3/v4.1 hoặc SMB 2.x/3.x
        ▼
┌─────────────────────────────────────────────────────┐
│              File Gateway (VM hoặc Hardware)         │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │              Local Cache (Cache Cục Bộ)        │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │  Recently Read/Written Files             │  │  │
│  │  │  (Tệp đọc/ghi gần đây — hot data)        │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────┘
                           │  HTTPS/TLS (port 443)
                           │  Mã hóa AES-256
                           ▼
                   ┌───────────────┐
                   │  Amazon S3   │
                   │   Bucket      │
                   │               │
                   │  file.txt → S3 object
                   │  Metadata → S3 object metadata
                   └───────────────┘
```

### Cách Hoạt Động Từng Bước

```
1. Ghi tệp:
   App ghi file.txt → NFS mount → File Gateway nhận
   ├── Ghi vào Local Cache ngay lập tức (low latency)
   ├── Xác nhận ghi thành công cho ứng dụng
   └── Async upload lên S3 (không block ứng dụng)

2. Đọc tệp (cache hit — có trong cache):
   App đọc file.txt → NFS mount → File Gateway kiểm tra cache
   └── Cache có → trả về từ local disk (milliseconds)

3. Đọc tệp (cache miss — không có trong cache):
   App đọc file.txt → NFS mount → File Gateway kiểm tra cache
   └── Cache miss → download từ S3 → lưu cache → trả về app
```

### Deployment Options (Tùy Chọn Triển Khai)

| Loại | Mô Tả | Khi Nào Dùng |
|------|--------|--------------|
| **VMware ESXi** | VM trên hypervisor | Datacenter có VMware |
| **Microsoft Hyper-V** | VM trên Hyper-V | Datacenter Windows |
| **KVM** | VM trên Linux KVM | Datacenter Linux open-source |
| **EC2 (trên AWS)** | Gateway trên EC2 | Kết nối giữa VPC và S3 |
| **Hardware Appliance** | Thiết bị phần cứng AWS | Không có ảo hóa, edge location |

**Yêu Cầu Tối Thiểu:**
- vCPU: 4 cores
- RAM: 16 GB
- Cache disk: Tối thiểu 150 GB (khuyến nghị SSD)
- Network: 1 Gbps hoặc hơn

---

## NFS vs SMB

### NFS — Network File System (Hệ Thống Tệp Mạng)

```
Đặc điểm:
├── Phiên bản hỗ trợ: NFSv3, NFSv4.1
├── Hệ điều hành: Linux, macOS, Unix
├── Authentication: IP-based hoặc IAM (khi bật)
└── Use cases: Linux workloads, containers, HPC

Cấu hình NFS mount trên Linux:
sudo mount -t nfs4 -o nfsvers=4.1 \
  <gateway-ip>:/mybucket/myshare /mnt/nfs-share

# hoặc trong /etc/fstab:
<gateway-ip>:/mybucket/myshare /mnt/nfs-share nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576 0 0
```

**Quyền Truy Cập NFS:**
- `Squash`: root squash (chuyển root → nobody), no squash
- `Client List`: danh sách IP được phép mount
- `Read-only` / `Read-write`: quyền đọc/ghi

### SMB — Server Message Block (Giao Thức Chia Sẻ Tệp Windows)

```
Đặc điểm:
├── Phiên bản hỗ trợ: SMB 2.x, SMB 3.x
├── Hệ điều hành: Windows, macOS
├── Authentication: Active Directory, guest access
└── Use cases: Windows file server, user home directories

Map network drive trên Windows:
net use Z: \\<gateway-ip>\myshare /user:domain\username password

# hoặc PowerShell:
New-PSDrive -Name "Z" -PSProvider FileSystem `
  -Root "\\<gateway-ip>\myshare" `
  -Credential (Get-Credential)
```

**Tích Hợp Active Directory:**
```
Cấu hình SMB với Active Directory (AD):
1. Join gateway vào AD domain
2. User authenticate bằng AD credentials
3. S3 object lưu với metadata của AD user
4. Audit trail: ai ghi/đọc file nào
```

### So Sánh NFS vs SMB

| Tiêu Chí | NFS | SMB |
|----------|-----|-----|
| OS tốt nhất | Linux/Unix | Windows |
| Authentication | IP-based / IAM | AD / Guest |
| Concurrent access | Tốt | Tốt |
| Windows ACL | Không hỗ trợ | Hỗ trợ đầy đủ |
| DFS (Distributed File System) | Không | Có (với AD) |

---

## Cache Cục Bộ

### Tầm Quan Trọng Của Cache

```
Không có cache tốt:
App ghi → chờ upload S3 → xác nhận → App tiếp tục
Độ trễ: 50-500ms (phụ thuộc băng thông)

Có cache tốt:
App ghi → cache (xác nhận ngay) → upload S3 async
Độ trễ ghi: < 5ms
```

### Sizing Cache (Tính Kích Thước Cache)

**Công Thức:**
```
Cache Size = Working Set Size × 1.2

Working Set = dữ liệu thực sự được truy cập thường xuyên
(không phải tổng dữ liệu trong S3)

Ví dụ:
- Tổng dữ liệu: 100 TB
- Working set (hot data): 500 GB (dữ liệu 30 ngày gần nhất)
- Cache cần: 500 GB × 1.2 = 600 GB SSD
- Chi phí cache: << Chi phí NAS 100 TB
```

**Loại Disk Khuyến Nghị:**
```
Write-heavy workload (nhiều ghi):
└── SSD cục bộ — tốc độ ghi cao, không là bottleneck

Read-heavy workload (nhiều đọc):
└── SSD hoặc NVMe — random read nhanh

Mixed workload:
└── SSD minimum — HDD có thể gây nghẽn cổ chai
```

### Cache Refresh (Làm Mới Cache)

```
File Gateway tự động refresh cache khi:
├── Client ghi tệp mới
├── Client mount lại share
└── Hết TTL (Time To Live — Thời Gian Sống) của metadata

Để đọc dữ liệu mới nhất từ S3 (tệp do S3 API ghi trực tiếp):
→ Dùng RefreshCache API để ép reload metadata từ S3
```

---

## Cấu Hình File Share

### Tạo File Share — Các Tham Số Quan Trọng

```yaml
# NFS File Share Configuration
NFSFileShare:
  GatewayARN: arn:aws:storagegateway:us-east-1:123456789012:gateway/sgw-12A3456B
  LocationARN: arn:aws:s3:::my-company-archive
  Role: arn:aws:iam::123456789012:role/StorageGatewayRole
  
  # Cache settings
  DefaultStorageClass: S3_STANDARD  # hoặc S3_STANDARD_IA, S3_INTELLIGENT_TIERING
  
  # Object metadata
  ObjectACL: private  # private, public-read, bucket-owner-full-control
  
  # NFS permissions
  Squash: RootSquash  # RootSquash, NoSquash, AllSquash
  ReadOnly: false
  ClientList:
    - 10.0.0.0/8    # Chỉ cho phép IP nội bộ
  
  # Encryption
  KMSEncrypted: true
  KMSKey: arn:aws:kms:us-east-1:123456789012:key/mrk-xxx
```

### S3 Object Mapping (Ánh Xạ Tệp → S3 Object)

```
File trên NFS/SMB          →    S3 Object Key
─────────────────────────────────────────────────────
/nfs-share/docs/report.pdf → mybucket/docs/report.pdf
/nfs-share/images/logo.png → mybucket/images/logo.png
/nfs-share/data/           → mybucket/data/ (prefix)

Metadata được lưu:
├── Content-Type (MIME type)
├── Last-Modified timestamp
├── File permissions (lưu trong user-defined metadata)
└── Ownership (uid/gid cho NFS)
```

---

## Use Cases Điển Hình

### 1. Media Production (Sản Xuất Media)

```
Vấn đề: Studio phim cần share storage 500TB cho:
- Editing workstations (máy dựng phim)
- Render farm (nông trại render)
- Archive thành phẩm

Giải pháp với File Gateway:
┌───────────────────────────────────────┐
│  On-Premises Studio                    │
│                                        │
│  Editing WS ──┐                        │
│  Render Farm ─┼── NFS ──► File Gateway │──► S3 (lưu trữ chính)
│  Director PC ─┘            │           │
│                     Local Cache 2TB    │
└───────────────────────────────────────┘

Kết quả:
- Tiết kiệm: Không cần mua thêm SAN/NAS đắt tiền
- Availability: 99.99% so với on-prem hardware
- DR: Tự động backup lên S3 cross-region
```

### 2. User Home Directories (Thư Mục Home Người Dùng)

```
Vấn đề: 5,000 nhân viên cần home directory
Yêu cầu: Mỗi user 20GB, tổng 100TB

Trước đây: NAS on-premises 100TB = $80,000
Với File Gateway + S3:
- File Gateway VM: $200/tháng
- S3 Standard-IA 100TB: $1,230/tháng
- Tổng: ~$1,430/tháng → ~$17,160/năm (tiết kiệm 78% vs NAS)

Cấu hình:
SMB Share với Active Directory
├── \\gateway\home\alice\ → s3://corp-home/alice/
├── \\gateway\home\bob\  → s3://corp-home/bob/
└── Permissions qua AD groups
```

### 3. Backup Off-site (Backup Ngoài Site)

```
Vấn đề: DR yêu cầu backup tại site khác
Chi phí site thứ hai: rất cao

Giải pháp:
Backup software (Veeam, CommVault) → SMB share → File Gateway → S3

Benefits:
- S3 durability: 99.999999999% (11 chín)
- S3 Cross-Region Replication tự động cho geo-redundancy
- S3 Object Lock — WORM cho compliance
- Không cần quản lý hardware backup
```

### 4. Data Lake Ingestion (Thu Nạp Dữ Liệu Vào Data Lake)

```
Vấn đề: ETL pipeline cần đưa dữ liệu từ on-prem lên S3

Legacy flow: On-prem → FTP → S3 (phức tạp, không reliable)
New flow: On-prem app ghi file → NFS → File Gateway → S3

Sau đó S3 trigger Lambda → AWS Glue Crawler → Athena query
→ Data lake hoàn chỉnh với minimal migration effort
```

---

## Hiệu Suất và Tối Ưu

### Yếu Tố Ảnh Hưởng Hiệu Suất

```
1. Băng thông kết nối Internet/Direct Connect
   ├── Tốt: Direct Connect 1Gbps+ cho production
   └── OK: Internet với 100Mbps+ cho backup

2. Cache Hit Rate (Tỷ Lệ Cache Trúng)
   ├── High cache hit (>80%) → latency thấp
   └── Low cache hit (<50%) → mỗi read phải fetch từ S3

3. File Size Pattern
   ├── Many small files (<1MB) → hiệu suất thấp do overhead
   └── Large files (>10MB) → hiệu suất tốt

4. Concurrent Connections (Kết Nối Đồng Thời)
   └── Mỗi gateway xử lý tốt đến hàng trăm connections
```

### Tối Ưu Cho Small Files

```bash
# Vấn đề: 1 triệu file nhỏ 10KB = 10GB nhưng ghi chậm
# Giải pháp: Batch → archive trước khi đẩy lên
tar -czf archive-$(date +%Y%m%d).tar.gz /data/small-files/
# Ghi archive vào NFS mount → 1 object lớn thay vì 1 triệu object nhỏ

# Hoặc dùng S3 Batch Operations sau khi upload
```

### CloudWatch Metrics Quan Trọng

```
GatewayName/CacheHitPercent
└── Mục tiêu: > 80%
└── Nếu thấp: tăng cache size hoặc xem xét working set

GatewayName/CacheUsed
└── Theo dõi để tránh cache đầy
└── Khi full: write performance giảm mạnh

GatewayName/ReadBytes / WriteBytes
└── Throughput thực tế
└── Dùng để size kết nối mạng

GatewayName/CloudBytesDownloaded
└── Số bytes download từ S3 về gateway
└── Cao = cache miss rate cao
```

---

## Bảo Mật

### Encryption (Mã Hóa)

```
In-transit (Khi Truyền):
├── Gateway ↔ S3: HTTPS/TLS tự động
└── App ↔ Gateway: NFS/SMB qua LAN (tin cậy internal network)

At-rest (Khi Lưu Trữ):
├── S3: SSE-S3 hoặc SSE-KMS (khuyến nghị KMS)
└── Local cache: mã hóa disk nếu dùng encrypted EBS/hardware

Cấu hình SSE-KMS cho file share:
KMSEncrypted: true
KMSKey: arn:aws:kms:region:account:key/key-id
```

### IAM Role Cho Gateway

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": [
        "arn:aws:s3:::my-gateway-bucket",
        "arn:aws:s3:::my-gateway-bucket/*"
      ]
    }
  ]
}
```

### Network Security (Bảo Mật Mạng)

```
Ports cần mở:
├── TCP 2049: NFS
├── TCP 445: SMB
├── TCP/UDP 111: NFS rpcbind
├── TCP 443: HTTPS về AWS
└── TCP 80: Gateway activation (chỉ 1 lần)

Best practices:
├── Đặt gateway trong DMZ hoặc security subnet riêng
├── Security Group chỉ cho phép IP on-prem cụ thể
├── VPC Endpoint cho S3 → không traffic qua internet
└── Audit log: S3 server access logging bật
```

---

## Chi Phí

### Thành Phần Chi Phí

```
1. Storage Gateway charges:
   - File Gateway: $0.01/GB data written to S3
   - Không tính phí dọc theo thời gian (không có gateway fee hàng tháng)

2. S3 Storage:
   - Phụ thuộc storage class được chọn
   - S3 Standard: $0.023/GB/tháng
   - S3 Standard-IA: $0.0125/GB/tháng
   - S3 Intelligent-Tiering: tự động tối ưu

3. Data Transfer:
   - Ghi lên S3 (upload): miễn phí
   - Đọc từ S3 về gateway: $0.09/GB (data transfer out)

4. S3 Requests:
   - PUT, COPY, POST, LIST: $0.005/1000 requests
   - GET, SELECT: $0.0004/1000 requests
```

### Ví Dụ Tính Chi Phí

```
Scenario: File server 10TB, 20% active monthly

File Gateway:
- 10TB × 20% × 0.01 = $20.48 data write fee

S3 Standard-IA (phù hợp vì không truy cập thường xuyên):
- 10TB × $0.0125 = $128/tháng

So với NAS on-prem 10TB:
- Hardware: ~$10,000 depreciated over 5 years = $167/tháng
- Maintenance/power/cooling: ~$100/tháng
- Total: ~$267/tháng

→ File Gateway + S3-IA: ~$148/tháng
→ Tiết kiệm: ~45% so với on-prem NAS
```

---

## Câu Hỏi Phỏng Vấn

**Q1: File Gateway là gì và tại sao cần nó?**

> File Gateway là một loại AWS Storage Gateway cung cấp giao diện NFS/SMB để ứng dụng on-premises có thể đọc/ghi vào Amazon S3 như thể đang dùng file server thông thường. Cần nó khi muốn:
> - Migrate dần lên cloud mà không phải viết lại ứng dụng
> - Tiết kiệm chi phí NAS/SAN on-premises
> - Backup dữ liệu tự động lên S3 với durability cao

**Q2: Giải thích cơ chế cache trong File Gateway.**

> File Gateway duy trì một local cache trên disk (SSD khuyến nghị) để:
> - Ghi: lưu vào cache trước → xác nhận cho app → upload S3 async (giảm latency ghi)
> - Đọc: hot data trong cache → trả về ngay (cache hit); cold data → fetch từ S3 → lưu cache
> Cache size nên bằng ~120% working set (dữ liệu thực sự dùng thường xuyên)

**Q3: File Gateway khác Storage Gateway loại Volume như thế nào?**

> File Gateway: giao thức NFS/SMB, object-level access trên S3, phù hợp file server
> Volume Gateway: giao thức iSCSI, block-level access, phù hợp database và ứng dụng cần block storage

**Q4: Làm sao tăng hiệu suất File Gateway?**

> 1. Tăng cache size (ưu tiên SSD)
> 2. Nâng cấp kết nối lên Direct Connect
> 3. Optimize file size (tránh millions of small files)
> 4. Monitor CacheHitPercent — target >80%
> 5. Dùng S3 Intelligent-Tiering để giảm chi phí tự động

**Q5: Khi nào KHÔNG nên dùng File Gateway?**

> - Workload cần POSIX compliance hoàn toàn (locking phức tạp) → dùng EFS
> - Database cần block storage với IOPS cao → dùng EBS + EC2
> - Migration một lần lớn → dùng DataSync (nhanh hơn, rẻ hơn)
> - Latency < 1ms → cache miss sẽ cao, không phù hợp

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
