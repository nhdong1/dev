# CRR — Cross-Region Replication — Sao Chép Liên Vùng

> CRR (Cross-Region Replication — Sao Chép Liên Vùng) tự động sao chép object từ S3 bucket ở một AWS Region sang bucket ở Region khác. Là nền tảng cho chiến lược Disaster Recovery (Khôi Phục Thảm Họa) và low-latency access (truy cập độ trễ thấp) toàn cầu.

## 📚 Mục Lục

1. [CRR Là Gì Và Hoạt Động Như Thế Nào?](#1-crr-là-gì-và-hoạt-động-như-thế-nào)
2. [Điều Kiện Bắt Buộc](#2-điều-kiện-bắt-buộc)
3. [Cấu Hình CRR](#3-cấu-hình-crr)
4. [Replication Time Control — RTC](#4-replication-time-control--rtc)
5. [Use Cases Thực Tế](#5-use-cases-thực-tế)
6. [Chi Phí](#6-chi-phí)
7. [Hạn Chế Và Lưu Ý Quan Trọng](#7-hạn-chế-và-lưu-ý-quan-trọng)
8. [CRR vs SRR — So Sánh](#8-crr-vs-srr--so-sánh)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. CRR Là Gì Và Hoạt Động Như Thế Nào?

### Luồng Dữ Liệu

```
                    AWS Region A (us-east-1)
                   ┌──────────────────────┐
                   │   Source Bucket      │
   Upload Object ──▶   (Bucket Nguồn)    │
                   │                      │
                   └──────────┬───────────┘
                              │
                    Asynchronous Replication
                    (Sao chép bất đồng bộ)
                    Thường trong vài giây
                              │
                   ┌──────────▼───────────┐
                   │ Destination Bucket   │
                   │ (Bucket Đích)        │
                   └──────────────────────┘
                    AWS Region B (eu-west-1)
```

### Đặc Điểm Cốt Lõi

- **Asynchronous (Bất Đồng Bộ):** Object được sao chép sau khi đã upload thành công ở nguồn
- **Automatic (Tự Động):** Không cần can thiệp thủ công, xảy ra khi có object mới/cập nhật
- **Secure (Bảo Mật):** Dữ liệu được mã hóa khi truyền qua TLS (Transport Layer Security — Bảo Mật Tầng Truyền Tải)
- **Granular (Chi Tiết):** Có thể lọc theo prefix, tag, kích thước

---

## 2. Điều Kiện Bắt Buộc

```
✅ PHẢI bật Versioning ở CẢ HAI bucket (nguồn và đích)
✅ Bucket nguồn và đích phải ở KHÁC Region (cho CRR)
✅ IAM Role phải có quyền:
   - s3:GetReplicationConfiguration (đọc cấu hình replication)
   - s3:ListBucket (liệt kê object)
   - s3:GetObjectVersionForReplication (đọc phiên bản object)
   - s3:ReplicateObject (ghi object sang đích)
   - s3:ReplicateDelete (sao chép hành động xóa)
✅ Nếu bucket đích dùng KMS encryption — IAM role cần quyền dùng KMS key đích
```

### Sơ Đồ IAM Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### IAM Permission Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::source-bucket"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::source-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateObjectTags"
      ],
      "Resource": "arn:aws:s3:::destination-bucket/*"
    }
  ]
}
```

---

## 3. Cấu Hình CRR

### 3.1 Qua AWS CLI

```bash
# Bước 1: Bật versioning ở bucket nguồn
aws s3api put-bucket-versioning \
  --bucket source-bucket \
  --versioning-configuration Status=Enabled

# Bước 2: Bật versioning ở bucket đích
aws s3api put-bucket-versioning \
  --bucket destination-bucket \
  --region eu-west-1 \
  --versioning-configuration Status=Enabled

# Bước 3: Tạo IAM role (dùng AWS Console hoặc CloudFormation)

# Bước 4: Áp dụng replication configuration
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json
```

### 3.2 File replication.json

```json
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "ReplicateEverything",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket",
        "StorageClass": "STANDARD_IA",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": { "Minutes": 15 }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": { "Minutes": 15 }
        },
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "arn:aws:kms:eu-west-1:123456789012:key/abcd1234"
        }
      },
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": {
          "Status": "Enabled"
        },
        "ReplicaModifications": {
          "Status": "Enabled"
        }
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
```

### 3.3 Qua Terraform

```hcl
resource "aws_s3_bucket_replication_configuration" "example" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  rule {
    id     = "ReplicateAll"
    status = "Enabled"

    filter {}

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"

      replication_time {
        status = "Enabled"
        time {
          minutes = 15
        }
      }

      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

### 3.4 Các Tùy Chọn Lọc Trong Rule

```json
"Filter": {
  "And": {
    "Prefix": "production/",
    "Tags": [
      { "Key": "replicate", "Value": "true" }
    ]
  }
}
```

---

## 4. Replication Time Control — RTC

RTC (Replication Time Control — Kiểm Soát Thời Gian Sao Chép) là tính năng **trả phí thêm** đảm bảo 99.99% object được replicate trong vòng **15 phút**.

### Khi Nào Cần RTC?

```
Không cần RTC: Hầu hết object replicate trong vài giây đến vài phút (best-effort)
Cần RTC:
  - DR với RTO (Recovery Time Objective) nghiêm ngặt
  - Regulatory compliance yêu cầu SLA cho replication
  - Active-active (hoạt động song song ở hai vùng) setup
```

### CloudWatch Metrics Khi Dùng RTC

Khi bật Replication Metrics và RTC, AWS tạo các CloudWatch metrics:

- **ReplicationLatency (Độ Trễ Sao Chép):** Thời gian tính từ khi PUT đến khi replicate xong
- **BytesPendingReplication (Byte Đang Chờ):** Tổng byte chưa được replicate
- **OperationsPendingReplication (Thao Tác Đang Chờ):** Số object đang chờ replicate

---

## 5. Use Cases Thực Tế

### Use Case 1: Disaster Recovery (Khôi Phục Thảm Họa)

```
Mục tiêu: RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) < 15 phút

Thiết lập:
- Primary Region: us-east-1
- DR Region: us-west-2
- Bật RTC (đảm bảo SLA 15 phút)
- Bật Delete Marker Replication: NO (không muốn xóa nhầm xóa cả DR)
- Storage Class ở đích: STANDARD_IA (tiết kiệm 46% vì ít truy cập)

Failover procedure (Quy trình chuyển đổi):
1. Route 53 — failover routing chuyển traffic sang DR
2. Application đọc/ghi từ bucket ở us-west-2
3. Khi primary phục hồi: S3 Batch Replication đồng bộ lại
```

### Use Case 2: Low Latency Access — Truy Cập Độ Trễ Thấp Toàn Cầu

```
Ứng dụng: CDN (Content Delivery Network) với user ở Mỹ và châu Âu

Thiết lập:
- us-east-1: Bucket chính, application ghi vào đây
- eu-west-1: Bucket replicate, user châu Âu đọc từ đây
- Không cần RTC (latency vài phút chấp nhận được với media content)
- Kết hợp với CloudFront origin group (nhóm nguồn) cho seamless failover

Lợi ích:
- Giảm latency từ ~150ms → ~20ms cho user châu Âu
- Tiết kiệm data transfer fee (phí truyền dữ liệu) qua CloudFront
```

### Use Case 3: Data Sovereignty — Tuân Thủ Chủ Quyền Dữ Liệu

```
Yêu cầu GDPR (General Data Protection Regulation — Quy định Bảo Vệ Dữ Liệu Chung):
- Dữ liệu của người dùng EU phải được lưu trong EU

Thiết lập:
- eu-west-1: Bucket chính cho dữ liệu EU
- CRR sang eu-central-1 (Frankfurt): Backup trong EU
- KHÔNG replicate sang us-east-1

Bucket Policy ngăn replication sang non-EU:
{
  "Condition": {
    "StringNotLike": {
      "aws:RequestedRegion": "eu-*"
    }
  },
  "Effect": "Deny",
  "Action": "s3:ReplicateObject"
}
```

### Use Case 4: Aggregated Logging — Tổng Hợp Nhật Ký

```
Mục tiêu: Thu thập log từ nhiều region về một nơi để phân tích

Thiết lập (Nhiều nguồn → Một đích):
- us-east-1 → central-log-bucket (us-east-1)
- eu-west-1 → central-log-bucket
- ap-southeast-1 → central-log-bucket

Dùng prefix khác nhau để phân biệt nguồn:
- Prefix filter: "region=us-east-1/"
- Prefix filter: "region=eu-west-1/"

Lưu ý: Đây là CRR (nhiều nguồn) + SRR (cùng region cũng có thể hội tụ)
```

---

## 6. Chi Phí

### Các Thành Phần Chi Phí

| Thành Phần | Giá |
| ---------- | ---- |
| Storage ở bucket đích | Theo storage class đích |
| Data transfer (Truyền Dữ Liệu) liên vùng | $0.02/GB (thay đổi theo region) |
| Request fee tại đích | $0.005/1,000 PUT |
| RTC (nếu bật) | $0.015/GB replicated + $0.03/1M object |

### Ví Dụ Tính Chi Phí

```
Scenario: Replicate 1TB/tháng từ us-east-1 sang eu-west-1

Storage đích (Standard-IA):  1,024GB × $0.0125 = $12.80
Data transfer:               1,024GB × $0.02   = $20.48
Request fee (ước 1M PUT):    1,000 × $0.005    = $5.00
                                              ────────
Tổng thêm mỗi tháng:                           $38.28

Chi phí nguồn (Standard):   1,024GB × $0.023 = $23.55
Tổng chi phí có CRR:        $23.55 + $38.28  = $61.83
```

---

## 7. Hạn Chế Và Lưu Ý Quan Trọng

### Object Đã Tồn Tại Trước Khi Bật CRR

```
❌ CRR KHÔNG tự động replicate object đã tồn tại
✅ Giải pháp: Dùng S3 Batch Replication (Sao Chép Hàng Loạt) để sync object cũ
```

### Delete Marker Replication

```
Mặc định: Delete markers KHÔNG được replicate
→ Xóa object ở nguồn KHÔNG xóa ở đích (an toàn cho DR)

Nếu bật DeleteMarkerReplication.Status = "Enabled":
→ Xóa ở nguồn sẽ tạo delete marker ở đích
→ Cẩn thận: xóa nhầm ở nguồn sẽ xóa cả DR!

Permanent deletions (xóa vĩnh viễn một phiên bản cụ thể):
→ KHÔNG BAO GIỜ được replicate, kể cả khi bật DeleteMarkerReplication
→ Đây là cơ chế bảo vệ intentional
```

### Replication Chỉ Một Chiều (Một Cấp)

```
Object được tạo ở bucket A → replicate sang bucket B ✅
Object được tạo ở bucket A → replicate sang B → B không replicate sang C ❌

Muốn chain: Phải cấu hình rule riêng B → C
Muốn bidirectional (hai chiều): Cấu hình hai rule riêng biệt A→B và B→A
```

### Ownership (Quyền Sở Hữu)

```
Mặc định: Object ở bucket đích thuộc sở hữu của account nguồn
→ Account đích không thể đọc được!

Giải pháp: Bật "Owner Override" trong replication rule
→ Object ở đích thuộc sở hữu account đích
```

---

## 8. CRR vs SRR — So Sánh

| Tiêu Chí | CRR (Liên Vùng) | SRR (Cùng Vùng) |
| -------- | --------------- | --------------- |
| Vị trí đích | Khác AWS Region | Cùng AWS Region |
| Mục đích chính | DR, low latency toàn cầu | Log aggregation, compliance |
| Data transfer fee | Có ($0.02/GB) | Không |
| Latency replication | Phụ thuộc vào khoảng cách vùng | Thấp hơn |
| Tuân thủ chủ quyền dữ liệu | Phức tạp hơn | Đơn giản hơn |

---

## 9. Câu Hỏi Phỏng Vấn

**Q1: Thiết kế DR với RPO < 15 phút dùng S3 như thế nào?**

> Bật CRR từ primary region sang DR region với RTC (Replication Time Control) enabled. RTC đảm bảo SLA 99.99% object được replicate trong 15 phút. Không bật Delete Marker Replication để tránh xóa nhầm xóa cả DR. Kết hợp Route 53 Failover Routing để tự động chuyển traffic.

**Q2: Object upload trước khi bật CRR có được replicate không?**

> Không. CRR chỉ replicate object MỚI tạo sau khi cấu hình rule. Để replicate object cũ, dùng S3 Batch Replication — tạo một Batch Operations job chạy `ReplicateObject` trên toàn bộ object trong bucket.

**Q3: Tại sao object ở bucket đích mặc định không thể đọc bởi account đích?**

> Mặc định, khi replicate sang bucket ở account khác, ownership (quyền sở hữu) của object vẫn thuộc account nguồn. Account đích không có quyền đọc. Giải pháp là bật "Replica Ownership Override" trong replication rule, hoặc thêm bucket policy cho phép account nguồn được `PutObject` và ACL ghi là `bucket-owner-full-control`.

**Q4: CRR có sao chép metadata và tags không?**

> Có, CRR sao chép object, metadata, và tags. Nếu bật `ReplicaModifications: Enabled`, thay đổi metadata hoặc tags ở object bản sao cũng được đồng bộ ngược lại bucket nguồn (two-way metadata sync).

**Q5: Một bucket có thể replicate sang nhiều đích không?**

> Có, một bucket có thể có nhiều replication rule với nhiều đích (destination) khác nhau. Mỗi rule có thể dùng filter (prefix/tag) khác nhau để phân luồng dữ liệu. Giới hạn là 1,000 rule trên một bucket.

---

## 📋 Checklist Triển Khai CRR

- [ ] Bật Versioning ở bucket nguồn
- [ ] Tạo bucket đích ở region khác với Versioning bật
- [ ] Tạo IAM role với đúng permissions
- [ ] Cân nhắc có cần RTC không (chi phí thêm nhưng đảm bảo SLA)
- [ ] Quyết định về Delete Marker Replication (thường tắt cho DR)
- [ ] Bật Owner Override nếu replicate cross-account
- [ ] Chạy Batch Replication để sync object cũ (nếu cần)
- [ ] Thiết lập CloudWatch alarm cho ReplicationLatency và BytesPendingReplication
- [ ] Test failover procedure thực tế

---

**Tiếp Theo:** [3-same-region-replication.md](./3-same-region-replication.md) — SRR và use cases
