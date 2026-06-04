# AWS Storage Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về các dịch vụ lưu trữ AWS

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/storage/
├── README.md                                [BẮT ĐẦU TẠI ĐÂY] Lộ trình học & tổng quan
├── INDEX.md                                 Chỉ mục này — toàn bộ file & trạng thái
│
├── 01-s3-fundamentals/
│   ├── README.md                            Tổng quan S3, storage classes, concepts cơ bản
│   ├── 1-bucket-and-object-model.md           Bucket, object, key, metadata, ETag
│   ├── 2-storage-classes.md                   Standard, IA, Glacier, Intelligent-Tiering — so sánh
│   ├── 3-versioning-and-mfa-delete.md         Quản lý phiên bản, xóa có xác thực 2FA
│   ├── 4-presigned-urls.md                    URL có chữ ký tạm thời — cách tạo và dùng
│   └── 5-multipart-upload.md                  Tải lên nhiều phần cho file lớn (>5GB)
│
├── 02-s3-advanced/
│   ├── README.md                            S3 nâng cao — replication, events, batch
│   ├── 1-lifecycle-policies.md                Chính sách vòng đời — tự động hóa chuyển tầng
│   ├── 2-cross-region-replication.md          CRR — Sao chép liên vùng, cấu hình, use cases
│   ├── 3-same-region-replication.md           SRR — Sao chép cùng vùng, compliance
│   ├── 4-s3-object-lock.md                    WORM — Write Once Read Many, Governance vs Compliance
│   ├── 5-s3-event-notifications.md            Thông báo sự kiện — Lambda, SQS, SNS, EventBridge
│   └── 6-s3-batch-operations.md               Thao tác hàng loạt — copy, tag, restore, invoke Lambda
│
├── 03-ebs/
│   ├── README.md                            EBS — Elastic Block Store, tổng quan
│   ├── 1-volume-types.md                      gp2/gp3, io1/io2, st1, sc1 — so sánh chi tiết
│   ├── 2-snapshots-and-lifecycle.md           Ảnh chụp nhanh, DLM — Data Lifecycle Manager
│   ├── 3-ebs-multi-attach.md                  Gắn kết nhiều EC2 — io1/io2 chỉ
│   ├── 4-performance-tuning.md                IOPS, Throughput, Latency — tối ưu hiệu suất
│   ├── 5-encryption-with-kms.md               Mã hóa với KMS — Key Management Service
│   └── 6-ebs-vs-instance-store.md             So sánh lưu trữ bền vững vs tạm thời
│
├── 04-efs/
│   ├── README.md                            EFS — Elastic File System, tổng quan
│   ├── 1-mount-targets-and-access-points.md   NFS mount targets, access points, IAM authz
│   ├── 2-performance-modes.md                 General Purpose vs Max I/O — khi nào dùng
│   ├── 3-throughput-modes.md                  Bursting, Provisioned, Elastic — so sánh
│   ├── 4-efs-intelligent-tiering.md           Phân tầng thông minh — Standard ↔ IA tự động
│   └── 5-efs-vs-ebs-vs-s3.md                  Bảng so sánh đầy đủ — chọn dịch vụ phù hợp
│
├── 05-security/
│   ├── README.md                            Bảo mật lưu trữ — tổng quan
│   ├── 1-iam-policies-for-storage.md          IAM policies cho S3, EBS, EFS — examples
│   ├── 2-s3-bucket-policies.md                Bucket policies, condition keys, cross-account
│   ├── 3-block-public-access.md               Chặn truy cập công khai — cấu hình & best practices
│   ├── 4-encryption-at-rest.md                SSE-S3, SSE-KMS, SSE-C, CSE — so sánh chi tiết
│   ├── 5-encryption-in-transit.md             TLS/HTTPS enforcement, VPC endpoints
│   ├── 6-vpc-endpoints.md                     Gateway endpoint vs Interface endpoint cho S3/EFS
│   └── 7-s3-access-analyzer.md                Phân tích quyền truy cập — phát hiện public access
│
├── 06-cost-optimization/
│   ├── README.md                            Tối ưu chi phí lưu trữ — tổng quan
│   ├── 1-s3-pricing-guide.md                  Bảng giá S3 — storage, requests, data transfer
│   ├── 2-ebs-pricing-guide.md                 Bảng giá EBS — gp3 vs io2 vs st1 — tính toán
│   ├── 3-lifecycle-rules-design.md            Thiết kế lifecycle rules — ví dụ thực tế
│   ├── 4-intelligent-tiering-when.md          Khi nào Intelligent-Tiering tiết kiệm chi phí
│   ├── 5-cost-allocation-tags.md              Thẻ phân bổ chi phí — phân tích theo team/project
│   └── 6-s3-storage-lens.md                   Storage Lens dashboard — phân tích toàn tổ chức
│
├── 07-monitoring/
│   ├── README.md                            Monitoring & Observability cho AWS storage
│   ├── 1-cloudwatch-s3-metrics.md             CloudWatch metrics cho S3 — BucketSizeBytes, etc.
│   ├── 2-cloudwatch-ebs-metrics.md            EBS metrics — VolumeReadOps, BurstBalance, Queue
│   ├── 3-cloudwatch-efs-metrics.md            EFS metrics — PermittedThroughput, BurstCreditBalance
│   ├── 4-s3-server-access-logging.md          Nhật ký truy cập S3 — cấu hình và phân tích
│   ├── 5-cloudtrail-for-storage.md            CloudTrail audit trail cho S3 API calls
│   └── 6-cost-anomaly-detection.md            Phát hiện bất thường chi phí — cảnh báo tự động
│
├── 08-disaster-recovery/
│   ├── README.md                            Disaster Recovery — RPO/RTO cho lưu trữ
│   ├── 1-rpo-rto-for-storage.md               RPO/RTO — ý nghĩa và thiết kế cho storage
│   ├── 2-s3-crr-for-dr.md                     Cross-Region Replication cho DR — thiết kế
│   ├── 3-ebs-snapshot-strategy.md             Chiến lược snapshot EBS — tần suất, retention
│   ├── 4-aws-backup-service.md                AWS Backup — backup tập trung, vault lock
│   └── 5-backup-vault-lock.md                 Vault lock — WORM cho backup, compliance
│
├── 09-storage-gateway/
│   ├── README.md                            Storage Gateway & Hybrid Cloud — tổng quan
│   ├── 1-file-gateway.md                      File Gateway — NFS/SMB → S3, use cases
│   ├── 2-volume-gateway.md                    Volume Gateway — iSCSI block storage → S3
│   ├── 3-tape-gateway.md                      Tape Gateway — Virtual Tape Library thay thế băng vật lý4-
│   ├── 4-datasync.md                          DataSync — đồng bộ tự động on-premises ↔ AWS5-
│   └── 5-direct-connect-vs-vpn.md             Direct Connect vs VPN cho hybrid storage
│
├── 10-snow-family/
│   ├── README.md                            Snow Family & di chuyển dữ liệu quy mô lớn
│   ├── 1-snowcone.md                          Snowcone — 8TB, nhỏ gọn, edge computing
│   ├── 2-snowball-edge.md                     Snowball Edge — 80TB storage / 42TB compute
│   ├── 3-snowmobile.md                        Snowmobile — 100PB, container, datacenter migration
│   ├── 4-snow-vs-datasync.md                  Chọn Snow Family hay DataSync — decision tree
│   └── 5-opshub-management.md                 OpsHub — giao diện quản lý Snow devices
│
├── 11-fsx/
│   ├── README.md                            FSx — Advanced File Systems, tổng quan
│   ├── 1-fsx-for-windows.md                   FSx Windows — Active Directory, SMB, DFS
│   ├── 2-fsx-for-lustre.md                    FSx Lustre — HPC, ML training, tích hợp S3
│   ├── 3-fsx-for-netapp-ontap.md              FSx NetApp ONTAP — enterprise migration, dedup
│   ├── 4-fsx-for-openzfs.md                   FSx OpenZFS — snapshots tức thì, clones
│   └── 5-fsx-comparison.md                    So sánh FSx variants — chọn đúng loại
│
├── 12-interview-prep/
│   ├── README.md                            Tổng quan chuẩn bị phỏng vấn AWS storage
│   ├── 1-INTERVIEW_GUIDE.md                   Top 20 câu hỏi phỏng vấn AWS storage — Q&A đầy đủ
│   ├── 2-system-design-scenarios.md           Tình huống thiết kế: data lake, backup, DR
│   ├── 3-star-stories.md                      Template câu chuyện STAR — storage incidents
│   ├── 4-cost-optimization-questions.md       Câu hỏi về tối ưu chi phí lưu trữ
│   └── 5-90-day-study-plan.md                 Kế hoạch học 90 ngày có cấu trúc
│
├── GLOSSARY.md                              (Sẽ tạo) Thuật ngữ AWS storage từ A–Z
├── RESOURCES.md                             (Sẽ tạo) Sách, blog, khóa học, tools
└── CHECKLIST.md                             (Sẽ tạo) Checklist trước phỏng vấn & production
```

---

## ✅ Đã Tạo / Đang Triển Khai

| Chủ Đề                      | File                            | Trạng Thái | Chất Lượng     |
| --------------------------- | ------------------------------- | ---------- | -------------- |
| **Tổng Quan & Lộ Trình**    | README.md                       | ✅         | Toàn diện      |
| **Chỉ Mục Đầy Đủ**          | INDEX.md                        | ✅         | Toàn diện      |
| **S3 Fundamentals**         | 01-s3-fundamentals/ (6 files)   | ✅         | Hoàn thành     |
| **S3 Nâng Cao**             | 02-s3-advanced/ (7 files)       | ✅         | Hoàn thành     |
| **EBS**                     | 03-ebs/ (7 files)               | ✅         | Hoàn thành     |
| **EFS**                     | 04-efs/ (6 files)               | ✅         | Hoàn thành     |
| **Bảo Mật**                 | 05-security/ (8 files)          | ✅         | Hoàn thành     |
| **Tối Ưu Chi Phí**          | 06-cost-optimization/ (7 files) | ✅         | Hoàn thành     |
| **Monitoring**              | 07-monitoring/ (7 files)        | ✅         | Hoàn thành     |
| **Disaster Recovery**       | 08-disaster-recovery/ (5 files) | ✅         | Hoàn thành     |
| **Storage Gateway**         | 09-storage-gateway/ (6 files)   | ✅         | Hoàn thành     |
| **Snow Family**             | 10-snow-family/ (6 files)       | ✅         | Hoàn thành     |
| **FSx**                     | 11-fsx/ (6 files)               | ✅         | Hoàn thành     |
| **Phỏng Vấn**               | 12-interview-prep/ (6 files)    | ✅         | Hoàn thành     |

---

## 🎯 Ưu Tiên Tạo (Theo Thứ Tự)

### Ưu Tiên Cao — Kỹ Năng Cốt Lõi

- [ ] `01-s3-fundamentals/README.md` — Nền tảng S3
- [ ] `01-s3-fundamentals/storage-classes.md` — Storage classes quan trọng nhất
- [ ] `05-security/README.md` — Bảo mật lưu trữ
- [ ] `05-security/encryption-at-rest.md` — Mã hóa — hỏi nhiều trong phỏng vấn
- [ ] `08-disaster-recovery/rpo-rto-for-storage.md` — RPO/RTO cho storage
- [ ] `12-interview-prep/INTERVIEW_GUIDE.md` — Chuẩn bị phỏng vấn

### Ưu Tiên Vừa — Kỹ Năng Vận Hành

- [ ] `03-ebs/volume-types.md` — EBS volume types
- [ ] `06-cost-optimization/lifecycle-rules-design.md` — Thiết kế lifecycle
- [ ] `02-s3-advanced/lifecycle-policies.md` — Lifecycle policies
- [ ] `07-monitoring/cloudwatch-ebs-metrics.md` — Monitoring EBS
- [ ] `04-efs/efs-vs-ebs-vs-s3.md` — Bảng so sánh quan trọng

### Ưu Tiên Thấp — Chủ Đề Chuyên Sâu

- [ ] `09-storage-gateway/file-gateway.md` — Hybrid cloud
- [ ] `10-snow-family/` — Migration tools
- [ ] `11-fsx/` — Enterprise file systems
- [ ] `GLOSSARY.md` — Thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu học

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học

```
1. Bắt đầu với README.md
2. Chọn lộ trình học (Beginner/Intermediate/Advanced)
3. Đi qua từng phần theo thứ tự
4. Thực hành trên AWS Free Tier hoặc lab
5. Xây dựng portfolio project
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào S3 (luôn hỏi) và Bảo Mật
3. Học 08-disaster-recovery/ (RPO/RTO luôn hỏi)
4. Chuẩn bị câu chuyện incident từ kinh nghiệm
5. Luyện tập giải thích với người khác (mock interview)
```

### Làm Việc Thực Tế

```
Dùng như tài liệu tham khảo:
- Trước deployment: Đọc 05-security/ checklist
- Incidents: Tra 07-monitoring/ để diagnosis
- Cost review: Theo 06-cost-optimization/
- DR planning: Theo 08-disaster-recovery/
- Hybrid setup: Theo 09-storage-gateway/
```

### System Design

```
1. Đọc 04-efs/efs-vs-ebs-vs-s3.md để chọn dịch vụ
2. Dùng 08-disaster-recovery/ cho DR design
3. Theo 05-security/ cho security architecture
4. Dùng 06-cost-optimization/ cho cost estimate
```

---

## 📊 Ước Tính Thời Gian Học

| Phần                       | Thời Gian    | Độ Khó   | Ưu Tiên     |
| -------------------------- | ------------ | -------- | ----------- |
| S3 Fundamentals            | 4–6 giờ      | ⭐        | Bắt buộc    |
| S3 Nâng Cao                | 5–7 giờ      | ⭐⭐      | Bắt buộc    |
| EBS                        | 4–6 giờ      | ⭐⭐      | Bắt buộc    |
| EFS                        | 3–4 giờ      | ⭐⭐      | Bắt buộc    |
| Bảo Mật                    | 6–8 giờ      | ⭐⭐⭐    | Bắt buộc    |
| Tối Ưu Chi Phí             | 4–5 giờ      | ⭐⭐      | Nên làm     |
| Monitoring                 | 3–4 giờ      | ⭐⭐      | Nên làm     |
| Disaster Recovery          | 5–7 giờ      | ⭐⭐⭐    | Bắt buộc    |
| Storage Gateway            | 3–4 giờ      | ⭐⭐      | Nên làm     |
| Snow Family                | 2–3 giờ      | ⭐        | Tùy chọn    |
| FSx                        | 4–6 giờ      | ⭐⭐⭐    | Tùy chọn    |

**Tổng: 45–65 giờ để nắm vững AWS Storage Services**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Beginner — Mới Bắt Đầu (0–1 năm)

- [ ] Phân biệt S3, EBS, EFS
- [ ] Tạo và cấu hình S3 bucket cơ bản
- [ ] Hiểu storage classes S3
- [ ] Tạo EBS volume và attach EC2
- [ ] Bucket policy đơn giản

**Thời gian để thành thạo:** 1–2 tháng

### Intermediate — Trung Cấp (1–3 năm)

- [ ] Thiết kế lifecycle policies hiệu quả
- [ ] Cấu hình replication (CRR/SRR)
- [ ] Implement encryption toàn diện
- [ ] Monitoring storage performance
- [ ] Tối ưu chi phí lưu trữ

**Thời gian để thành thạo:** 2–3 tháng để nâng cao

### Advanced — Nâng Cao (3–5+ năm)

- [ ] Kiến trúc data lake quy mô lớn
- [ ] Multi-region DR strategy
- [ ] Storage Gateway hybrid cloud
- [ ] Enterprise FSx workloads
- [ ] Cost optimization quy mô lớn

**Thời gian để thành thạo:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Cần                              | Vị Trí                                                              |
| -------------------------------- | ------------------------------------------------------------------- |
| Tổng quan nhanh                  | [README.md](README.md)                                              |
| S3 storage classes               | [01-s3-fundamentals/storage-classes.md](01-s3-fundamentals/storage-classes.md) |
| Bảo mật bucket                   | [05-security/s3-bucket-policies.md](05-security/s3-bucket-policies.md) |
| So sánh EBS volume types         | [03-ebs/volume-types.md](03-ebs/volume-types.md)                    |
| EFS vs EBS vs S3                 | [04-efs/efs-vs-ebs-vs-s3.md](04-efs/efs-vs-ebs-vs-s3.md)           |
| Tối ưu chi phí                   | [06-cost-optimization/README.md](06-cost-optimization/README.md)    |
| Câu hỏi phỏng vấn                | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md) |
| DR planning                      | [08-disaster-recovery/README.md](08-disaster-recovery/README.md)    |
| Hybrid cloud                     | [09-storage-gateway/README.md](09-storage-gateway/README.md)        |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Storage — Tiến Độ Hoàn Thành

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] S3 bucket và object model
- [ ] Storage classes (6 loại)
- [ ] EBS volume types (5 loại)
- [ ] EFS cơ bản
- [ ] IAM cho storage

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)

- [ ] S3 lifecycle policies
- [ ] Cross-Region Replication
- [ ] Encryption at-rest và in-transit
- [ ] EBS performance tuning
- [ ] CloudWatch monitoring cho storage
- [ ] Disaster recovery design

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)

- [ ] Storage Gateway (File/Volume/Tape)
- [ ] Cost optimization strategies
- [ ] Security hardening đầy đủ
- [ ] FSx variants
- [ ] Snow Family

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Data lake architecture trên S3
- [ ] Multi-region storage strategy
- [ ] Enterprise storage patterns
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích sự khác biệt S3 vs EBS vs EFS không cần nhìn tài liệu
- [ ] Chọn đúng storage class cho mỗi use case
- [ ] Thiết kế IAM policy an toàn cho S3
- [ ] Chọn EBS volume type phù hợp cho workload

### ✅ Năng Lực Vận Hành

- [ ] Thiết kế lifecycle rules hiệu quả
- [ ] Implement DR strategy với RPO/RTO cụ thể
- [ ] Monitor và cảnh báo storage metrics
- [ ] Tối ưu chi phí một cách có hệ thống

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời top 20 câu hỏi AWS storage tự tin
- [ ] Kể 2–3 câu chuyện incident theo STAR format
- [ ] Thiết kế kiến trúc lưu trữ cho hệ thống phức tạp
- [ ] Thảo luận về trade-offs và ràng buộc chi phí
- [ ] Giải thích deep-dive về bảo mật lưu trữ

---

## 💡 Mẹo Học Hiệu Quả

1. **Thực hành qua AWS Free Tier:** Tạo bucket, upload, test lifecycle — đừng chỉ đọc lý thuyết
2. **Hiểu nguyên lý, không chỉ thuộc tính năng:** Biết TẠI SAO S3 dùng eventual consistency
3. **Luyện tính toán chi phí:** Tự tính bill hàng tháng cho các scenario khác nhau
4. **Dùng AWS CLI hàng ngày:** Thao tác CLI nhanh hơn Console — quan trọng trong phỏng vấn
5. **Theo dõi AWS Storage Blog:** Tính năng mới ra liên tục — cập nhật hàng tháng
6. **Biết khi nào KHÔNG dùng S3:** Instance Store cho temp data, EBS cho latency thấp
7. **Test backup thực sự:** Restore S3 object và EBS snapshot để biết RTO thực tế
8. **Document incident thực tế:** Post-mortem là cơ hội học hỏi tốt nhất

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Đóng góp được hoan nghênh:

- [ ] Sửa lỗi trong nội dung hiện tại
- [ ] Thêm phần cho chủ đề chưa có
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn cho khái niệm phức tạp
- [ ] Hướng dẫn dịch vụ mới của AWS

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 2.2 (Toàn bộ 12 topics hoàn thành — kể cả 12-interview-prep/)
**Trạng Thái:** ✅ README.md | ✅ INDEX.md | ✅ 01-s3-fundamentals/ | ✅ 02-s3-advanced/ | ✅ 03-ebs/ | ✅ 04-efs/ | ✅ 05-security/ | ✅ 06-cost-optimization/ | ✅ 07-monitoring/ | ✅ 08-disaster-recovery/ | ✅ 09-storage-gateway/ | ✅ 10-snow-family/ | ✅ 11-fsx/ | ✅ 12-interview-prep/
