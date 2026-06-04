# 🏗️ System Design Scenarios — Thiết Kế Hệ Thống AWS Storage

> Các bài toán thiết kế hệ thống thực tế thường gặp trong phỏng vấn senior/architect. Mỗi scenario có context rõ ràng, các constraint cần xét đến, và thiết kế đầy đủ kèm giải thích trade-offs.

---

## 🎯 Cách Tiếp Cận System Design Interview

Khi interviewer đưa một bài toán, đừng code ngay. Theo framework sau:

```
1. CLARIFY (2-3 phút): Hỏi rõ yêu cầu — scale, budget, team size
2. ESTIMATE (2-3 phút): Tính toán sơ bộ — data volume, request rate
3. DESIGN HIGH-LEVEL (5-7 phút): Vẽ kiến trúc tổng quan
4. DEEP DIVE (10-15 phút): Chi tiết thành phần quan trọng
5. TRADE-OFFS (3-5 phút): Thảo luận alternatives và giới hạn
```

---

## Scenario 1: Hệ Thống Data Lake Cho Startup E-Commerce

### Bối Cảnh

Bạn là Senior Engineer tại một startup e-commerce đang tăng trưởng nhanh:
- 500.000 đơn hàng/ngày
- Log từ 50 microservices — mỗi service sinh ~1GB log/ngày
- Cần phân tích: báo cáo doanh thu real-time, phân tích hành vi user, fraud detection
- Budget: $5.000/tháng cho storage và analytics
- Team: 3 data engineers

### Câu Hỏi Clarification Quan Trọng

```
- Dữ liệu cũ cần giữ bao lâu? (5 năm vì tax compliance)
- Latency của query? (Dashboard business: <30s, fraud detection: <1s)
- Ai cần access? (Data analysts, ML engineers, compliance team)
- Data có PII — Personally Identifiable Information (Thông Tin Định Danh Cá Nhân) không? (Có — địa chỉ, SĐT)
```

### Ước Tính Dữ Liệu

```
Log data: 50 services × 1 GB/ngày = 50 GB/ngày = 1.5 TB/tháng = 18 TB/năm
Order data: 500.000 orders × 2KB/order = 1 GB/ngày
User events (clickstream): 500.000 users × 100 events × 1KB = 50 GB/ngày
Tổng: ~100 GB/ngày = 3 TB/tháng
5 năm: ~180 TB
```

### Kiến Trúc Thiết Kế

```
┌─────────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER (Tầng Thu Thập)               │
├──────────────┬─────────────────┬───────────────────────────────┤
│  Microservices│  Kinesis        │  Direct API calls             │
│  (Fluentd)   │  Data Firehose  │  (từ applications)            │
│              │  (Đường ống     │                               │
│              │  Dữ liệu)       │                               │
└──────┬───────┴────────┬────────┴──────────────────┬────────────┘
       │                │                            │
       ▼                ▼                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    RAW ZONE (Tầng Thô)                          │
│    s3://company-raw/{service}/{year}/{month}/{day}/{hour}/       │
│                                                                 │
│  Storage Class: Standard → IA (30 ngày) → Glacier (90 ngày)    │
│  Encryption: SSE-KMS (vì có PII)                               │
│  Lifecycle: Glacier Deep Archive sau 365 ngày                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼ (AWS Glue ETL — Extract Transform Load)
┌─────────────────────────────────────────────────────────────────┐
│                   SILVER ZONE (Tầng Xử Lý)                      │
│    s3://company-silver/{domain}/{year}/{month}/{day}/            │
│                                                                 │
│  - PII đã được mask/tokenize (ẩn danh hóa)                     │
│  - Format: Parquet (columnar — dạng cột, nén tốt hơn)          │
│  - Partition: theo domain + thời gian                           │
│  Storage Class: Intelligent-Tiering                             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼ (Aggregation jobs)
┌─────────────────────────────────────────────────────────────────┐
│                    GOLD ZONE (Tầng Phân Tích)                   │
│    s3://company-gold/reports/{type}/{year}/{month}/             │
│                                                                 │
│  - Business metrics đã aggregate (tổng hợp)                    │
│  - Format: Parquet hoặc JSON                                    │
│  Storage Class: Standard (query thường xuyên)                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
            ┌─────────────┴─────────────┐
            ▼                           ▼
   ┌─────────────────┐       ┌──────────────────────┐
   │  Amazon Athena  │       │  Amazon QuickSight   │
   │  (SQL query     │       │  (Dashboard BI       │
   │  trực tiếp S3)  │       │  — Business          │
   │                 │       │  Intelligence)       │
   └─────────────────┘       └──────────────────────┘
```

### Quyết Định Thiết Kế Quan Trọng

**Tại sao Parquet thay vì JSON/CSV?**
- Columnar format — chỉ đọc columns cần thiết → Athena query rẻ hơn 10x
- Nén tốt hơn → giảm 70-80% storage size
- Athena tính phí theo data scanned — ít data = ít tiền

**Tại sao SSE-KMS thay vì SSE-S3?**
- Có PII → cần audit trail ai đọc dữ liệu (KMS CloudTrail log)
- Compliance: GDPR, PDPA (Luật Bảo Vệ Dữ Liệu Cá Nhân) yêu cầu biết ai truy cập PII

**Partition strategy:**
```
Không tốt: s3://raw/logs/app.log (scan toàn bộ)
Tốt hơn:   s3://raw/service=payment/year=2026/month=05/day=16/
Athena sẽ chỉ scan partition cần thiết → giảm 95% cost query
```

### Tính Toán Chi Phí Ước Tính

```
Storage Raw (5 năm, với lifecycle):
  Năm 1-3: 108TB × $0.023 = $2,484
  Năm 4-5: Glacier $0.004/GB = $432
  
Athena queries: 100TB scanned/tháng × $5/TB = $500/tháng
  (giảm xuống $50 với Parquet partitioning)

Glue ETL: 100 DPU-hours/ngày × $0.44 = $44/ngày = $1,320/tháng

Tổng: ~$2,000-3,000/tháng → trong budget $5,000
```

---

## Scenario 2: Media Streaming Platform — Video On Demand

### Bối Cảnh

Nền tảng xem phim trực tuyến (giống Netflix nhỏ):
- 10 triệu users
- 50.000 video, mỗi video ~2GB (nhiều resolution)
- 1 triệu views/ngày
- Cần upload video, xử lý, phân phối toàn cầu
- SLA — Service Level Agreement (Thỏa Thuận Mức Dịch Vụ): 99.9% availability, <3s start time

### Kiến Trúc

```
UPLOAD FLOW (Luồng Tải Lên):
Creator → Presigned PUT URL → S3 Raw Upload bucket
→ S3 Event Notification → Lambda → start MediaConvert job
→ MediaConvert transcode sang HLS/DASH (multi-bitrate)
→ Output lưu vào S3 CDN Origin bucket

PLAYBACK FLOW (Luồng Phát):
User → CloudFront → S3 Origin (nếu cache miss)
CloudFront cache: Edge Locations toàn cầu
                  TTL: 30 ngày cho video segments
```

**Lưu trữ:**

| Loại Dữ Liệu | Storage Class | Lý Do |
|--------------|---------------|-------|
| Raw upload (video gốc) | Standard → Glacier (30 ngày) | Chỉ cần trong quá trình xử lý |
| Transcoded video (phát sóng) | Standard | Luôn cần sẵn sàng |
| Video cũ >1 năm ít xem | Intelligent-Tiering | Không dự đoán được ai xem lại |
| Metadata, thumbnails | Standard | Nhỏ, truy cập thường xuyên |

**Tại sao dùng Presigned PUT URL thay vì upload qua server?**
```
Upload qua server: Creator → Server → S3  (server phải xử lý băng thông video lớn)
Presigned PUT URL: Creator → S3 trực tiếp (server không chịu tải bandwidth)
→ Giảm 100% server bandwidth cost cho upload
→ Upload nhanh hơn (trực tiếp đến S3 edge)
```

**Tối ưu CloudFront:**
- Origin Shield: Một lớp cache trung gian → giảm request lên S3
- Signed Cookies: Chỉ user có subscription mới xem được (không dùng Presigned URL vì phải sign từng segment)

---

## Scenario 3: Multi-Region Disaster Recovery cho Fintech

### Bối Cảnh

Công ty Fintech xử lý thanh toán:
- **RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục):** 0 — không được mất bất kỳ transaction nào
- **RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục):** 5 phút
- Primary region: `ap-southeast-1` (Singapore)
- DR region: `ap-east-1` (Hong Kong)
- Regulatory: MAS — Monetary Authority of Singapore (Cơ Quan Tiền Tệ Singapore) yêu cầu data backup ở Việt Nam region hoặc offshore

### Kiến Trúc DR cho Storage

```
PRIMARY REGION (ap-southeast-1):
├── RDS Aurora (PostgreSQL) — transaction DB chính
│   └── Aurora Global Database → replicate sang DR (<1 giây)
├── S3 Bucket: transaction-logs-primary
│   └── CRR — Cross-Region Replication → transaction-logs-dr (ap-east-1)
│       └── Replication time: <15 phút (SLA của S3 RTC — Replication Time Control)
├── EBS Volumes (EC2 app servers)
│   └── Snapshot mỗi 1 giờ → copy sang ap-east-1 tự động
└── AWS Backup Vault (ap-southeast-1)
    └── Backup Vault Copy → ap-east-1

DR REGION (ap-east-1):
├── Aurora Global Database secondary cluster (read-only, failover ready)
├── S3 Bucket: transaction-logs-dr
│   └── Object Lock Compliance (7 năm) — không ai xóa được
├── Pre-warmed EC2 (stopped) — warm standby
└── AWS Backup Vault (ap-east-1)
```

### RPO = 0 Cho Transaction Data

```
Không thể đạt RPO = 0 với snapshot-based backup.
Giải pháp:
  1. Aurora Global Database: synchronous replication với RPO <1 giây
  2. S3 CRR với Replication Time Control (RTC):
     - S3 RTC đảm bảo 99.99% objects replicated trong 15 phút
     - Monitoring: CloudWatch metric ReplicationLatency
  3. Application-level dual-write (Ghi song song):
     - Application ghi vào S3 primary VÀ DR cùng lúc
     - Phức tạp hơn nhưng RPO thực sự = 0
```

### Runbook Failover (Sổ Tay Chuyển Sang DR)

```
T+0   : Phát hiện sự cố primary (CloudWatch alarm)
T+1   : Tự động trigger Route 53 failover routing
T+2   : Aurora Global Database promote secondary → primary
T+3   : EC2 instances ở DR region start (pre-warmed)
T+5   : Health check pass → traffic chuyển sang DR
RTO đạt được: 5 phút ✓
```

### S3 Object Lock Cho Compliance

Transaction logs cần giữ 7 năm (MAS requirement):
```
→ S3 Object Lock, Compliance Mode, 7 năm retention
→ Kể cả incident response team cũng không xóa được trong compliance period
→ Audit trail đầy đủ qua CloudTrail
```

---

## Scenario 4: Shared File Storage Cho Container Workloads

### Bối Cảnh

Công ty SaaS chạy Kubernetes trên EKS:
- 200 pods cần shared storage cho uploaded files (PDF, Excel)
- Files upload bởi user, cần xử lý và serve lại
- Team dev muốn dùng `PersistentVolumeClaim — PVC (Yêu Cầu Volume Cố Định)` tiêu chuẩn Kubernetes

### So Sánh Các Giải Pháp

| Giải Pháp | Ưu | Nhược | Phù Hợp Khi |
|-----------|-----|-------|-------------|
| **EFS + EFS CSI Driver** | Native NFS, POSIX-compliant | Latency cao hơn EBS | File sharing giữa pods |
| **EBS (gp3)** | Latency thấp | Chỉ 1 pod (ReadWriteOnce) | Database, stateful apps |
| **S3 + s3fs hoặc Mountpoint** | Rẻ nhất, scale tốt | Không full POSIX | Object access pattern |

**Khuyến nghị cho bài toán này: EFS**

```
Lý do:
- 200 pods cần access đồng thời → EFS hỗ trợ ReadWriteMany (RWX)
- EBS không support multi-pod write
- S3 không có POSIX semantics (file lock, directory ops)

Cấu hình EFS:
- Throughput Mode: Elastic (tự scale theo workload)
- Performance Mode: General Purpose (latency thấp nhất)
- Access Points: tạo access point riêng cho mỗi tenant (isolation)
- Encryption: EFS default encryption (KMS)

Kubernetes:
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes: ["ReadWriteMany"]  ← chỉ EFS và FSx hỗ trợ
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
```

**EFS Access Points cho Multi-Tenant:**
```
Tenant A → /data/tenant-a/ (access point enforce path và user)
Tenant B → /data/tenant-b/
→ Mỗi tenant chỉ thấy thư mục của mình
→ Không cần application-level isolation phức tạp
```

---

## Scenario 5: Hybrid Storage — On-Premises + AWS

### Bối Cảnh

Bệnh viện với 20 năm data y tế:
- 500TB data cũ trên NAS — Network Attached Storage (Lưu Trữ Đính Kèm Mạng) on-premises
- Cần migrate lên cloud nhưng giữ on-premises system hoạt động
- Regulatory: Data không được rời Việt Nam → phải dùng AWS region tại Singapore hoặc ap-southeast-1
- Kết nối: 1Gbps đường truyền dedicated

### Kiến Trúc Hybrid

```
ON-PREMISES:
├── EMR — Electronic Medical Records (Hồ Sơ Y Tế Điện Tử) system
├── NAS với 500TB historical data
└── Storage Gateway Appliance (VM hoặc hardware)

AWS:
├── S3 (ap-southeast-1) — lưu trữ cloud
├── Direct Connect (1Gbps dedicated)
└── AWS Backup

MIGRATION PHASES:
Phase 1 — Gateway (Hiện tại):
  Cài File Gateway trên NAS
  → File Gateway present NFS share cho on-prem apps
  → Dữ liệu write được cache local và sync lên S3 tự động
  → Apps không biết đang dùng S3 — transparent

Phase 2 — DataSync (3 tháng):
  AWS DataSync agent on-premises
  → Sync 500TB historical data lên S3 (incremental)
  → Kiểm tra checksum từng file

Phase 3 — Cloud-native (6 tháng):
  Apps chuyển dần sang đọc từ S3 trực tiếp
  → File Gateway chỉ còn cho legacy apps chưa migrate
```

**Tại sao DataSync thay vì copy thủ công?**
```
- DataSync: 10Gbps throughput (dùng nhiều parallel streams)
- Tự kiểm tra checksum — đảm bảo data integrity
- Monitor tiến trình qua CloudWatch
- 500TB / 1Gbps = ~4.5 ngày (vs vài tuần nếu copy thủ công)
```

---

## 📝 Checklist Khi Trả Lời System Design

Trước khi kết thúc câu trả lời, đảm bảo đã đề cập:

- [ ] **Scale:** Data volume, request rate, growth rate
- [ ] **Availability:** Multi-AZ, Multi-Region nếu cần
- [ ] **Durability:** Backup, replication strategy
- [ ] **Security:** Encryption, access control, audit
- [ ] **Cost:** Storage class selection, lifecycle rules, rough estimate
- [ ] **Monitoring:** CloudWatch alarms, S3 Storage Lens
- [ ] **Trade-offs:** Giải thích tại sao chọn giải pháp này, không phải alternatives
- [ ] **Evolution path:** Hệ thống sẽ scale như thế nào khi grow 10x?

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
