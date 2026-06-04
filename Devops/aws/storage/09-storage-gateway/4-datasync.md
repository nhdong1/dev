# 🔄 AWS DataSync — Đồng Bộ Dữ Liệu Tự Động On-Premises ↔ AWS

> AWS DataSync (Đồng Bộ Dữ Liệu AWS) là dịch vụ truyền dữ liệu trực tuyến được quản lý hoàn toàn, tự động hóa việc di chuyển và đồng bộ dữ liệu giữa on-premises storage và AWS storage services (S3, EFS, FSx), với tốc độ lên đến 10 Gbps và tích hợp kiểm tra tính toàn vẹn dữ liệu tự động.

---

## 📚 Mục Lục

1. [DataSync vs Storage Gateway](#datasync-vs-storage-gateway)
2. [Kiến Trúc DataSync](#kiến-trúc-datasync)
3. [DataSync Agent](#datasync-agent)
4. [Locations và Tasks](#locations-và-tasks)
5. [Filtering và Scheduling](#filtering-và-scheduling)
6. [Hiệu Suất](#hiệu-suất)
7. [Tính Toàn Vẹn Dữ Liệu](#tính-toàn-vẹn-dữ-liệu)
8. [Use Cases Điển Hình](#use-cases-điển-hình)
9. [Bảo Mật](#bảo-mật)
10. [Chi Phí](#chi-phí)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DataSync vs Storage Gateway

### Khi Nào Dùng Loại Nào

```
AWS DataSync:
├── Di chuyển dữ liệu MỘT LẦN (one-time migration)
├── Đồng bộ định kỳ (hourly/daily/weekly batch sync)
├── Tốc độ cao tối ưu: dùng hết băng thông có sẵn
├── Use case: migration, disaster recovery data seeding
└── Tư duy: "chuyển/copy dữ liệu"

AWS Storage Gateway:
├── Hybrid storage liên tục (ongoing)
├── Ứng dụng dùng cloud storage như local storage
├── Low latency cho real-time reads/writes
├── Use case: file server, backup target, block storage
└── Tư duy: "mount cloud như local disk"
```

### So Sánh Bảng

| Tiêu Chí | DataSync | Storage Gateway |
|----------|----------|-----------------|
| **Mục đích** | Transfer/Sync data | Hybrid storage |
| **Mô hình** | Batch/Scheduled | Continuous/Real-time |
| **Giao thức nguồn** | NFS, SMB, HDFS, S3, EFS, FSx | NFS, SMB, iSCSI, VTL |
| **Giao thức đích** | S3, EFS, FSx, Azure Blob | S3, S3 Glacier |
| **Tốc độ** | Maximize throughput | Optimize latency |
| **Verification** | Checksum tự động | Không có |
| **Offline support** | Không | Stored Mode có |

---

## Kiến Trúc DataSync

```
On-Premises / Other Cloud Source
        │
        │  NFS / SMB / HDFS
        ▼
┌─────────────────────────────────────────────────────┐
│         DataSync Agent (VM on-premises)              │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Task Executor                                  │  │
│  │  ├── Parallel scan source (quét nguồn song song)│  │
│  │  ├── Checksum generation (tạo checksum)         │  │
│  │  ├── Compression (nén dữ liệu)                  │  │
│  │  ├── Encryption in-flight                       │  │
│  │  └── Multi-threaded transfer (truyền đa luồng)  │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────┘
                           │ TLS 1.2 (HTTPS, port 443)
                           │ Tối ưu TCP cho WAN
                           ▼
             ┌─────────────────────────────────┐
             │    AWS DataSync Service          │
             │    (quản lý bởi AWS)             │
             └──────────────┬──────────────────┘
                            │
             ┌──────────────┼───────────────────┐
             ▼              ▼                   ▼
        ┌─────────┐   ┌──────────┐    ┌──────────────┐
        │ Amazon  │   │ Amazon   │    │  Amazon FSx   │
        │   S3    │   │   EFS    │    │  (Windows,   │
        │         │   │          │    │   Lustre,    │
        └─────────┘   └──────────┘    │   ONTAP)     │
                                      └──────────────┘
```

### Multi-Region và Cross-Account

```
DataSync hỗ trợ:
├── Same account, same region: on-prem → S3 same region
├── Cross-region: on-prem → S3 us-east-1 sang eu-west-1
├── Cross-account: sync giữa 2 AWS accounts khác nhau
└── Cloud-to-cloud: S3 → EFS, EFS → FSx, etc.

Ví dụ cross-account sync:
Account A (Production) ──DataSync──► Account B (Archive)
                              └── Isolation: account B không có IAM access Production
```

---

## DataSync Agent

### Triển Khai Agent

```
Agent = VM chạy on-premises, kết nối source storage và AWS

Deployment options:
├── VMware ESXi: OVA image
├── KVM: QCOW2 image
├── Microsoft Hyper-V: VHD image
└── AWS EC2: Amazon Linux AMI (khi source là trong VPC)

Yêu cầu tối thiểu:
├── vCPU: 4 cores
├── RAM: 32 GB
├── Disk: 80 GB cho OS
└── Network: 10 Gbps NIC (khuyến nghị để maximize throughput)
```

### Agent Activation (Kích Hoạt Agent)

```bash
# Bước 1: Deploy agent VM
# Bước 2: Truy cập agent local UI: http://<agent-ip>
# Bước 3: Lấy activation key từ UI
# Bước 4: Kích hoạt qua AWS CLI hoặc Console

aws datasync create-agent \
  --activation-key "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX" \
  --agent-name "on-prem-agent-01" \
  --tags Key=Environment,Value=Production
```

### Agent Monitoring

```
CloudWatch metrics cho agent:
├── AgentOnline: agent có kết nối về AWS không
└── Nếu offline → báo động → kiểm tra mạng/agent

Bảo trì:
└── Agent tự động update software khi kết nối AWS
   → Không cần manual patch
```

---

## Locations và Tasks

### Location (Vị Trí)

```
Location = endpoint (điểm cuối) nguồn hoặc đích

Source locations (Vị Trí Nguồn):
├── NFS: on-prem NFS server
├── SMB: on-prem Windows file share
├── HDFS: Hadoop Distributed File System
├── Amazon S3: bucket trong cùng hoặc khác account
├── Amazon EFS: EFS file system
├── Amazon FSx: Windows, Lustre, ONTAP, OpenZFS
├── Azure Blob Storage (cross-cloud!)
└── Google Cloud Storage (cross-cloud!)

Destination locations (Vị Trí Đích):
├── Amazon S3 (tất cả storage classes)
├── Amazon EFS
└── Amazon FSx (tất cả variants)
```

```bash
# Tạo NFS location (nguồn on-prem)
aws datasync create-location-nfs \
  --server-hostname "192.168.1.100" \
  --subdirectory "/data/archive" \
  --on-prem-config AgentArns=arn:aws:datasync:region:account:agent/agent-xxx

# Tạo S3 location (đích)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::my-archive-bucket \
  --subdirectory /2024/archive \
  --s3-config BucketAccessRoleArn=arn:aws:iam::account:role/DataSyncRole \
  --s3-storage-class STANDARD_IA
```

### Task (Tác Vụ)

```
Task = kết hợp source location + destination location + config

aws datasync create-task \
  --source-location-arn arn:aws:datasync:...:location/loc-xxx \
  --destination-location-arn arn:aws:datasync:...:location/loc-yyy \
  --name "NightlyArchiveSync" \
  --options '{
    "VerifyMode": "ONLY_FILES_TRANSFERRED",
    "OverwriteMode": "ALWAYS",
    "Atime": "BEST_EFFORT",
    "Mtime": "PRESERVE",
    "Uid": "INT_VALUE",
    "Gid": "INT_VALUE",
    "PreserveDeletedFiles": "REMOVE",
    "PreserveDevices": "NONE",
    "PosixPermissions": "PRESERVE",
    "BytesPerSecond": -1,
    "TaskQueueing": "ENABLED",
    "LogLevel": "TRANSFER"
  }'
```

### Task Options Quan Trọng

| Option | Mô Tả | Giá Trị Phổ Biến |
|--------|--------|------------------|
| **VerifyMode** | Kiểm tra checksum sau transfer | `ONLY_FILES_TRANSFERRED` hoặc `POINT_IN_TIME_CONSISTENT` |
| **OverwriteMode** | Xử lý file đã tồn tại ở đích | `ALWAYS` (overwrite) hoặc `NEVER` |
| **PreserveDeletedFiles** | Xử lý file bị xóa ở nguồn | `PRESERVE` hoặc `REMOVE` |
| **Mtime** | Giữ nguyên modification timestamp | `PRESERVE` |
| **BytesPerSecond** | Giới hạn bandwidth | `-1` (không giới hạn) hoặc số bytes |

---

## Filtering và Scheduling

### Include/Exclude Filters (Bộ Lọc)

```bash
# Chỉ sync file .log
aws datasync create-task ... \
  --includes FilterType=SIMPLE_PATTERN,Value="*.log"

# Loại trừ thư mục temp và file tạm
aws datasync create-task ... \
  --excludes '[
    {"FilterType": "SIMPLE_PATTERN", "Value": "*/tmp/*"},
    {"FilterType": "SIMPLE_PATTERN", "Value": "*.tmp"},
    {"FilterType": "SIMPLE_PATTERN", "Value": "*.swp"}
  ]'
```

### Scheduling (Lên Lịch)

```bash
# Chạy mỗi đêm lúc 2 giờ sáng UTC
aws datasync update-task-execution \
  --task-arn arn:aws:datasync:region:account:task/task-xxx \
  --schedule ScheduleExpression="cron(0 2 * * ? *)"

# Chạy mỗi giờ
aws datasync update-task-execution \
  --schedule ScheduleExpression="cron(0 * * * ? *)"

# Chạy thủ công
aws datasync start-task-execution \
  --task-arn arn:aws:datasync:region:account:task/task-xxx
```

---

## Hiệu Suất

### Tại Sao DataSync Nhanh

```
DataSync tối ưu cho WAN (Wide Area Network — Mạng Diện Rộng):

1. Parallel scan (Quét Song Song):
   Nhiều luồng đồng thời quét và index source file system
   → Khởi động transfer nhanh hơn rsync đơn luồng

2. Multi-threaded transfer (Truyền Đa Luồng):
   Nhiều kết nối TCP song song thay vì 1 kết nối
   → Maximize bandwidth utilization (tận dụng băng thông)

3. Adaptive compression (Nén Thích Ứng):
   Nén dữ liệu in-flight
   → Giảm lượng data thực sự truyền qua mạng

4. Incremental transfer (Truyền Tăng Dần):
   Chỉ transfer files mới hoặc thay đổi
   → Tái đồng bộ (re-sync) sau lần đầu rất nhanh

Tốc độ thực tế:
├── Qua internet 1Gbps: ~100 MB/s
├── Direct Connect 10Gbps: ~1 GB/s
└── So với rsync qua internet: DataSync nhanh hơn 10x
```

### Bandwidth Throttling (Giới Hạn Băng Thông)

```bash
# Giới hạn 100 MB/s để không ảnh hưởng production traffic
aws datasync update-task \
  --task-arn arn:aws:datasync:...:task/task-xxx \
  --options BytesPerSecond=104857600  # 100 MB/s

# Giờ hành chính: 50 MB/s; ban đêm: không giới hạn
# → Dùng 2 schedules: daytime throttled + nighttime unlimited
```

---

## Tính Toàn Vẹn Dữ Liệu

### Verification Modes (Chế Độ Xác Minh)

```
DataSync tự động kiểm tra data integrity:

ONLY_FILES_TRANSFERRED (mặc định):
└── Checksum chỉ cho files vừa transfer
   → Nhanh hơn, đủ cho hầu hết use cases

POINT_IN_TIME_CONSISTENT:
└── Checksum toàn bộ destination
   → Chậm hơn nhưng đảm bảo hoàn toàn
   → Dùng cho final migration cutover

NONE:
└── Không kiểm tra
   → Nhanh nhất, chỉ dùng khi data không quan trọng
```

### Checksum Process

```
Với mỗi file transferred:
1. Tính SHA-256/MD5 checksum phía nguồn trước khi transfer
2. Transfer file qua TLS
3. Tính checksum phía đích sau khi nhận
4. So sánh: nếu không khớp → retry tự động
5. Nếu vẫn lỗi → ghi vào error log, tiếp tục file khác

→ DataSync cam kết không có bit flipping (lỗi bit) sau transfer
```

---

## Use Cases Điển Hình

### 1. One-time Migration (Di Chuyển Một Lần)

```
Scenario: Migration 500TB data từ NAS on-prem lên S3

Plan:
Phase 1 (Bulk Transfer — bulk sync):
├── DataSync agent → NAS → S3
├── Chạy liên tục 24/7 qua Direct Connect 10Gbps
└── Ước tính: 500TB / (1GB/s × 86400s) ≈ 5.8 ngày

Phase 2 (Catch-up Sync — đồng bộ bù):
├── Chạy incremental sync mỗi giờ
└── Chỉ sync files thay đổi trong thời gian bulk chạy

Phase 3 (Final Cutover — chuyển đổi cuối):
├── Stop writes trên NAS (maintenance window)
├── Chạy final DataSync với POINT_IN_TIME_CONSISTENT
├── Verify tổng số files và bytes
└── Update app config trỏ sang S3

Risk: Rất thấp vì data exists cả 2 nơi cho đến cutover
```

### 2. Hybrid Backup Sync

```
Scenario: Backup files trên NAS cần sync lên S3 Glacier-IA hàng đêm

Task config:
├── Source: NFS 192.168.1.100:/backup
├── Destination: S3 s3://company-backup-archive/ GLACIER_IR
├── Schedule: cron(0 1 * * ? *) — 1AM daily
├── OverwriteMode: ALWAYS
├── PreserveDeletedFiles: PRESERVE (không xóa S3 khi NAS xóa)
└── BytesPerSecond: -1 (dùng hết bandwidth ban đêm)

Lợi ích:
├── Offsite copy tự động mà không cần tape
├── S3 Glacier Instant Retrieval: restore trong milliseconds
└── Chi phí: ~$0.004/GB/tháng (x10 rẻ hơn S3 Standard)
```

### 3. Cross-Region Data Replication

```
Scenario: Replicate S3 bucket từ us-east-1 sang eu-west-1
Mục đích: GDPR compliance, local copy tại EU

Lưu ý: S3 CRR (Cross-Region Replication) thường tốt hơn cho ongoing
→ DataSync phù hợp cho initial seeding của CRR

DataSync task:
├── Source: S3 location us-east-1
├── Destination: S3 location eu-west-1
├── Không cần agent (cloud-to-cloud không cần agent)
└── Sau khi seed xong → enable S3 CRR cho ongoing sync
```

### 4. EFS → S3 Archival (Lưu Trữ)

```
Scenario: EFS file system 50TB tăng nhanh, cần giảm chi phí
EFS Standard: $0.30/GB/tháng
S3 Standard-IA: $0.0125/GB/tháng → tiết kiệm 96%

Setup DataSync:
├── Source: EFS file system
├── Destination: S3 Standard-IA
├── Filter: chỉ files không access trong 90 ngày
├── Schedule: cron(0 3 * * 0 *) — Chủ Nhật 3AM
└── PreserveDeletedFiles: PRESERVE (backup, không phải sync)

Kết quả: Tiết kiệm ~$14,375/tháng trên 50TB
```

---

## Bảo Mật

### Encryption (Mã Hóa)

```
In-transit:
└── TLS 1.2 tự động từ agent về AWS
   → Không cần cấu hình gì thêm

At-rest tại đích:
├── S3: SSE-S3 hoặc SSE-KMS (cấu hình trong S3 destination)
├── EFS: KMS encryption tại EFS level
└── FSx: tùy loại FSx

Agent authentication với AWS:
└── IAM role assigned to agent (không dùng access keys)
```

### IAM Permissions cho DataSync

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListMultipartUploadParts",
        "s3:PutObjectTagging",
        "s3:GetObjectTagging",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-datasync-bucket",
        "arn:aws:s3:::my-datasync-bucket/*"
      ]
    }
  ]
}
```

### Network Security

```
Ports cần mở (từ agent ra internet):
├── TCP 443: HTTPS về AWS DataSync service
├── TCP 443: HTTPS về CloudWatch, CloudTrail
└── Không cần inbound ports

VPC Endpoint cho DataSync:
└── Tạo Interface VPC Endpoint → DataSync traffic không qua internet
   Endpoint: com.amazonaws.region.datasync
```

---

## Chi Phí

### Pricing Model

```
DataSync charge theo:
└── $0.0125/GB dữ liệu đã copy (data copied)

Không tính phí cho:
├── Agent software (miễn phí)
├── Số lượng tasks
├── Số lần chạy
└── CloudWatch monitoring

Ngoài ra còn:
├── S3/EFS/FSx storage tại đích
├── Data transfer in (upload lên S3): miễn phí
└── AWS Direct Connect (nếu dùng): riêng biệt
```

### Ví Dụ Chi Phí

```
Scenario 1: One-time migration 100TB
DataSync: 100,000 GB × $0.0125 = $1,250 (chạy một lần)

Scenario 2: Daily incremental sync 100GB/ngày
DataSync: 100 GB × $0.0125 × 30 ngày = $37.50/tháng
→ Cộng S3 storage: 100TB S3-IA × $12.5 = $1,250/tháng

So với thuê nhân lực manual transfer:
0.25 FTE × $80,000/năm = $1,667/tháng
→ DataSync: $37.50/tháng → tiết kiệm 97%
```

---

## Câu Hỏi Phỏng Vấn

**Q1: DataSync khác Storage Gateway thế nào? Khi nào dùng cái nào?**

> - **DataSync**: tối ưu cho batch transfer/sync, maximize throughput, có verification. Dùng cho migration, scheduled sync.
> - **Storage Gateway**: tối ưu cho ongoing hybrid operations, low latency, mount như local storage. Dùng cho ứng dụng cần cloud storage liên tục.
> Rule of thumb: "Di chuyển/copy data" → DataSync. "Mount cloud như local disk" → Storage Gateway.

**Q2: DataSync đảm bảo data integrity như thế nào?**

> DataSync tự động tính checksum (SHA-256) phía nguồn trước khi transfer và kiểm tra lại phía đích sau khi nhận. Nếu không khớp, tự động retry. Tùy chọn `POINT_IN_TIME_CONSISTENT` kiểm tra toàn bộ destination — nên dùng cho final cutover migration.

**Q3: Làm sao migrate 500TB từ on-prem lên S3 với downtime tối thiểu?**

> 1. Cài DataSync agent, tạo task, bật initial bulk sync (chạy 5-7 ngày)
> 2. Chạy incremental sync mỗi giờ để catch up
> 3. Khi gần cutover: maintenance window ngắn (1-2 giờ), chạy final sync với POINT_IN_TIME_CONSISTENT
> 4. Verify, cập nhật app config, done. Downtime chỉ là maintenance window.

**Q4: DataSync có thể transfer giữa 2 AWS services không (không cần on-prem agent)?**

> Có. DataSync hỗ trợ cloud-to-cloud transfer: S3 → EFS, S3 → FSx, EFS → S3, etc. Không cần agent, toàn bộ chạy trong AWS network. Ví dụ: sync S3 us-east-1 → S3 eu-west-1 để seed dữ liệu trước khi enable CRR.

**Q5: Làm thế nào tối ưu chi phí DataSync cho data lớn?**

> 1. Compress at source trước khi sync (DataSync charge theo bytes transferred)
> 2. Filter chỉ sync files cần thiết (exclude logs tạm, tmp files)
> 3. Schedule ban đêm để dùng bandwidth rẻ hơn (Direct Connect off-peak)
> 4. Dùng S3 Standard-IA hoặc Glacier ở đích để giảm storage cost
> 5. Bật incremental mode (chỉ sync changed files) sau lần initial

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
