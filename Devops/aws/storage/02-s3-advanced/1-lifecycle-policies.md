# S3 Lifecycle Policies — Chính Sách Vòng Đời

> Lifecycle Policy (Chính Sách Vòng Đời) là cơ chế tự động chuyển object giữa các storage class hoặc xóa object sau một khoảng thời gian xác định — giảm chi phí lưu trữ mà không cần can thiệp thủ công.

## 📚 Mục Lục

1. [Lifecycle Policy Là Gì?](#1-lifecycle-policy-là-gì)
2. [Transition Actions — Hành Động Chuyển Tầng](#2-transition-actions--hành-động-chuyển-tầng)
3. [Expiration Actions — Hành Động Hết Hạn](#3-expiration-actions--hành-động-hết-hạn)
4. [Quy Tắc Chuyển Tầng Hợp Lệ](#4-quy-tắc-chuyển-tầng-hợp-lệ)
5. [Cấu Hình Qua AWS Console & CLI](#5-cấu-hình-qua-aws-console--cli)
6. [Ví Dụ Thực Tế Theo Use Case](#6-ví-dụ-thực-tế-theo-use-case)
7. [Chi Phí & Lưu Ý Quan Trọng](#7-chi-phí--lưu-ý-quan-trọng)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Lifecycle Policy Là Gì?

Lifecycle Policy là tập hợp các **rule (quy tắc)** định nghĩa:

- **Khi nào** chuyển object sang storage class rẻ hơn
- **Khi nào** xóa object hoặc phiên bản cũ của object

### Tại Sao Cần Lifecycle Policy?

```
Vấn đề thực tế:
- Log files được tạo mỗi ngày → truy cập nhiều trong 7 ngày → ít dần sau 30 ngày → không bao giờ xem sau 1 năm
- Backup snapshots cần giữ lại 7 năm theo quy định nhưng hiếm khi truy cập
- Media uploads: ảnh/video truy cập nhiều lúc mới upload, ít dần theo thời gian

Giải pháp:
- Tự động hạ tầng lưu trữ theo chu kỳ sống của dữ liệu
- Tiết kiệm 60–80% chi phí so với để tất cả ở Standard mãi
```

### Kiến Trúc Tổng Quan

```
Upload Object
      │
      ▼
[S3 Standard]  ─── 30 ngày ──▶ [Standard-IA]  ─── 60 ngày ──▶ [Glacier Instant]
                                                                        │
                                                               90 ngày thêm
                                                                        ▼
                                                              [Glacier Deep Archive]
                                                                        │
                                                               1825 ngày (5 năm)
                                                                        ▼
                                                                   [Xóa]
```

---

## 2. Transition Actions — Hành Động Chuyển Tầng

Transition Actions (Hành Động Chuyển Tầng) tự động di chuyển object sang storage class rẻ hơn sau N ngày.

### Storage Classes Và Chi Phí So Sánh

| Storage Class | Chi Phí Lưu Trữ | Retrieval Fee | Thời Gian Lấy Data | Minimum Duration |
| ------------- | --------------- | ------------- | ------------------- | ---------------- |
| Standard | $0.023/GB | $0 | Ngay lập tức | Không |
| Standard-IA | $0.0125/GB | $0.01/GB | Ngay lập tức | **30 ngày** |
| One Zone-IA | $0.01/GB | $0.01/GB | Ngay lập tức | **30 ngày** |
| Glacier Instant | $0.004/GB | $0.03/GB | Mili-giây | **90 ngày** |
| Glacier Flexible | $0.0036/GB | $0.01/GB | Phút đến giờ | **90 ngày** |
| Glacier Deep Archive | $0.00099/GB | $0.02/GB | 12–48 giờ | **180 ngày** |

> **Minimum Duration (Thời Gian Lưu Tối Thiểu):** Ngay cả khi xóa sớm hơn, bạn vẫn bị tính phí cho khoảng thời gian tối thiểu này.

### Thứ Tự Chuyển Tầng Hợp Lệ

```
Standard ──→ Standard-IA ──→ One Zone-IA ──→ Glacier Instant ──→ Glacier Flexible ──→ Deep Archive
         └──────────────────────────────────→ Glacier Instant ──→ Glacier Flexible ──→ Deep Archive
         └──────────────────────────────────────────────────────→ Glacier Flexible ──→ Deep Archive
         └──────────────────────────────────────────────────────────────────────────→ Deep Archive
```

**Lưu ý quan trọng:**
- Không thể đi ngược chiều (không thể chuyển từ Glacier về Standard qua lifecycle)
- Muốn "nâng cấp" phải restore rồi copy sang bucket mới

---

## 3. Expiration Actions — Hành Động Hết Hạn

Expiration Actions (Hành Động Hết Hạn) tự động xóa object sau một khoảng thời gian.

### Các Loại Expiration

#### 3.1 Object Expiration (Hết Hạn Object)

Xóa object hiện tại sau N ngày kể từ ngày tạo.

```json
{
  "Expiration": {
    "Days": 365
  }
}
```

#### 3.2 Noncurrent Version Expiration (Hết Hạn Phiên Bản Cũ)

Khi Versioning được bật, xóa các phiên bản không phải phiên bản hiện tại sau N ngày.

```json
{
  "NoncurrentVersionExpiration": {
    "NoncurrentDays": 90,
    "NewerNoncurrentVersions": 3
  }
}
```

- `NoncurrentDays`: Số ngày giữ phiên bản cũ
- `NewerNoncurrentVersions`: Giữ lại tối đa N phiên bản mới nhất (bỏ qua cái cũ hơn ngay lập tức)

#### 3.3 Incomplete Multipart Upload Cleanup (Dọn Dẹp Tải Lên Dở Dang)

Xóa các phần của Multipart Upload (Tải Lên Nhiều Phần) bị bỏ dở sau N ngày.

```json
{
  "AbortIncompleteMultipartUpload": {
    "DaysAfterInitiation": 7
  }
}
```

Nếu không cấu hình, các phần dở dang vẫn chiếm dung lượng và bị tính phí.

#### 3.4 Expired Object Delete Markers (Xóa Marker Hết Hạn)

Khi object có Versioning và tất cả phiên bản đã bị xóa, còn lại một "delete marker" (điểm đánh dấu xóa) trống. Có thể tự động dọn dẹp.

```json
{
  "Expiration": {
    "ExpiredObjectDeleteMarker": true
  }
}
```

---

## 4. Quy Tắc Chuyển Tầng Hợp Lệ

### Ràng Buộc Thời Gian

```
Standard → Standard-IA: Tối thiểu 30 ngày ở Standard
Standard-IA → Glacier Instant: Tối thiểu 30 ngày ở Standard-IA (90 ngày tổng từ upload)
Glacier Instant → Glacier Flexible: Không giới hạn thêm
Glacier Flexible → Deep Archive: Không giới hạn thêm
```

### Lọc Object Theo Filter (Bộ Lọc)

Mỗi rule có thể áp dụng cho:

```json
"Filter": {
  "And": {
    "Prefix": "logs/",
    "Tags": [
      { "Key": "department", "Value": "engineering" }
    ],
    "ObjectSizeGreaterThan": 131072,
    "ObjectSizeLessThan": 10485760
  }
}
```

- **Prefix**: Áp dụng cho object có key bắt đầu bằng `logs/`
- **Tags**: Áp dụng cho object có tag cụ thể
- **ObjectSizeGreaterThan/LessThan**: Áp dụng theo kích thước (byte)

> **Quan trọng:** Standard-IA và One Zone-IA có phí tối thiểu 128KB. Object nhỏ hơn 128KB chuyển sang IA có thể **tốn hơn** so với để ở Standard. Dùng `ObjectSizeGreaterThan: 131072` để lọc.

---

## 5. Cấu Hình Qua AWS Console & CLI

### 5.1 Qua AWS CLI

```bash
# Xem lifecycle configuration hiện tại
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-company-data

# Đặt lifecycle configuration từ file JSON
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-company-data \
  --lifecycle-configuration file://lifecycle.json

# Xóa toàn bộ lifecycle configuration
aws s3api delete-bucket-lifecycle \
  --bucket my-company-data
```

### 5.2 File lifecycle.json Ví Dụ Đầy Đủ

```json
{
  "Rules": [
    {
      "ID": "MoveLogsToGlacier",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

### 5.3 Qua AWS CloudFormation

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-company-data
      LifecycleConfiguration:
        Rules:
          - Id: ArchiveLogs
            Status: Enabled
            Prefix: logs/
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER_IR
            ExpirationInDays: 2555
            NoncurrentVersionExpirationInDays: 90
            AbortIncompleteMultipartUpload:
              DaysAfterInitiation: 7
```

### 5.4 Qua Terraform

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "example" {
  bucket = aws_s3_bucket.example.id

  rule {
    id     = "ArchiveLogs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }

    expiration {
      days = 2555
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

---

## 6. Ví Dụ Thực Tế Theo Use Case

### Use Case 1: Application Logs

```
Yêu cầu: Giữ log 7 năm, truy cập nhanh trong 30 ngày đầu, sau đó archive

Rule:
- 0–30 ngày:   Standard          ($0.023/GB)
- 31–90 ngày:  Standard-IA       ($0.0125/GB)
- 91–365 ngày: Glacier Instant   ($0.004/GB)
- 366+ ngày:   Deep Archive      ($0.00099/GB)
- 2555 ngày:   Xóa

Tiết kiệm ước tính với 1TB log/tháng: ~70% so với giữ nguyên Standard
```

```json
{
  "Rules": [{
    "ID": "ApplicationLogs7Years",
    "Status": "Enabled",
    "Filter": { "Prefix": "app-logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR" },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 }
  }]
}
```

### Use Case 2: User Uploaded Media

```
Yêu cầu: Ảnh/video người dùng upload, truy cập nhiều tuần đầu, sau đó ít dần

Rule:
- 0–7 ngày:    Standard (truy cập nhiều)
- 8–180 ngày:  Standard-IA (truy cập thỉnh thoảng)
- 181+ ngày:   Glacier Instant (archive nhưng cần truy cập nhanh)
- Phiên bản cũ: Xóa sau 30 ngày
```

```json
{
  "Rules": [{
    "ID": "UserMediaArchive",
    "Status": "Enabled",
    "Filter": {
      "And": {
        "Prefix": "user-uploads/",
        "ObjectSizeGreaterThan": 131072
      }
    },
    "Transitions": [
      { "Days": 7,   "StorageClass": "STANDARD_IA" },
      { "Days": 180, "StorageClass": "GLACIER_IR" }
    ],
    "NoncurrentVersionExpiration": {
      "NoncurrentDays": 30
    },
    "AbortIncompleteMultipartUpload": {
      "DaysAfterInitiation": 3
    }
  }]
}
```

### Use Case 3: Database Backup

```
Yêu cầu: Backup daily, giữ 7 bản ngày, 4 bản tuần, 12 bản tháng, 5 bản năm
→ Phức tạp hơn, thường dùng kết hợp với AWS Backup

Rule đơn giản hóa:
- 0–30 ngày:   Standard-IA
- 31–90 ngày:  Glacier Instant
- 91–365 ngày: Glacier Flexible
- 365+ ngày:   Deep Archive
- 1825 ngày:   Xóa (5 năm)
```

### Use Case 4: Compliance Archive (Lưu Trữ Tuân Thủ)

```
Yêu cầu: Financial records, giữ 7 năm theo SEC rule 17a-4, không được xóa sớm

Cần kết hợp với Object Lock Compliance Mode:
- 0–30 ngày:   Standard-IA
- 31+ ngày:    Deep Archive
- Object Lock: Compliance Mode, 7 năm
```

---

## 7. Chi Phí & Lưu Ý Quan Trọng

### 7.1 Phí Chuyển Tầng (Transition Fee)

Mỗi lần object chuyển từ tầng này sang tầng khác, AWS tính phí request:

| Chuyển Từ → Đến | Phí |
| --------------- | ---- |
| Standard → Standard-IA | $0.01 / 1,000 object |
| Standard → Glacier | $0.05 / 1,000 object |
| Standard → Deep Archive | $0.05 / 1,000 object |

Nếu có hàng tỷ object nhỏ, phí chuyển tầng có thể lớn hơn tiết kiệm từ storage rẻ hơn!

### 7.2 Minimum Storage Duration Fee (Phí Lưu Tối Thiểu)

Nếu object bị xóa hoặc chuyển tầng trước thời gian tối thiểu, vẫn bị tính đủ phí:

```
Object 100GB lưu ở Standard-IA được 10 ngày rồi xóa:
→ Bị tính phí như lưu 30 ngày (vì minimum là 30 ngày)
→ 100GB × $0.0125/GB × 30 ngày = bị tính đủ
```

### 7.3 Small Object Problem (Vấn Đề Object Nhỏ)

Standard-IA tính phí tối thiểu 128KB mỗi object. Object 1KB lưu ở Standard-IA:
- Standard: 1KB × $0.023/GB = rất rẻ
- Standard-IA: 128KB × $0.0125/GB = **đắt hơn 128 lần**

**Giải pháp:** Dùng filter `ObjectSizeGreaterThan: 131072` (128KB = 131072 bytes).

### 7.4 Thời Gian Thực Thi

Lifecycle rules không chạy chính xác vào ngày N. AWS chạy lifecycle jobs một lần mỗi ngày, thường vào đầu ngày UTC. Object có thể tồn tại thêm vài giờ so với số ngày trong rule.

### 7.5 Intelligent-Tiering So Với Lifecycle

| Tiêu Chí | Lifecycle Policy | Intelligent-Tiering |
| -------- | ---------------- | ------------------- |
| Cách hoạt động | Rule cứng theo số ngày | Tự động theo access pattern |
| Phù hợp khi | Access pattern dự đoán được | Access pattern không biết trước |
| Chi phí quản lý | Không | $0.0025/1,000 object |
| Linh hoạt | Thấp (cần update rule nếu pattern thay đổi) | Cao |
| Phí retrieval | Có (IA, Glacier) | Không có |

---

## 8. Câu Hỏi Phỏng Vấn

**Q1: Sự khác biệt giữa Transition Actions và Expiration Actions?**

> Transition Actions di chuyển object sang storage class khác để giảm chi phí lưu trữ, object vẫn tồn tại. Expiration Actions xóa object (hoặc phiên bản cũ) hoàn toàn. Thường kết hợp cả hai: chuyển tầng để tiết kiệm, sau đó xóa khi hết thời gian retention.

**Q2: Tại sao object nhỏ hơn 128KB không nên chuyển sang Standard-IA?**

> Standard-IA tính phí tối thiểu cho 128KB mỗi object. Object 1KB lưu ở Standard-IA vẫn bị tính như 128KB, đắt hơn rất nhiều so với để ở Standard. Dùng filter `ObjectSizeGreaterThan` để chỉ chuyển tầng những object đủ lớn.

**Q3: Lifecycle rule có áp dụng cho object đã tồn tại trước khi tạo rule không?**

> Có, lifecycle rule áp dụng cho TẤT CẢ object trong bucket (hoặc theo prefix/tag trong filter), kể cả object đã upload trước đó. Thời gian tính từ ngày object được tạo (`CreationDate`), không phải từ ngày tạo rule.

**Q4: Làm sao debug khi lifecycle rule không hoạt động?**

> Kiểm tra: (1) Rule Status = "Enabled", (2) Filter prefix/tag đúng không, (3) Đủ số ngày chưa (lifecycle chạy một lần/ngày UTC), (4) Object size có đủ 128KB cho IA không, (5) Versioning enabled nếu có NoncurrentVersion rule. Dùng S3 Storage Lens để xem distribution theo storage class.

**Q5: Chi phí gì phát sinh khi object chuyển từ Standard sang Glacier?**

> Hai loại phí: (1) **Transition request fee** — $0.05 / 1,000 object, (2) **Glacier overhead** — mỗi object Glacier tốn thêm 32KB overhead cho metadata (tính theo Glacier price). Nếu archive hàng triệu file nhỏ, phí overhead có thể lớn hơn tiết kiệm từ storage rẻ hơn.

---

## 📋 Checklist Triển Khai

Trước khi triển khai lifecycle policy lên production:

- [ ] Tính toán chi phí hiện tại vs sau khi áp dụng rule
- [ ] Kiểm tra kích thước object trung bình — có nhỏ hơn 128KB không?
- [ ] Xác nhận access pattern — có phù hợp với thứ tự chuyển tầng không?
- [ ] Bật versioning nếu cần NoncurrentVersion rules
- [ ] Test với prefix cụ thể trước khi áp dụng toàn bucket
- [ ] Thêm `AbortIncompleteMultipartUpload` để tránh phí dữ liệu dở dang
- [ ] Monitor S3 Storage Lens sau 7 ngày để xác nhận rule hoạt động

---

**Tiếp Theo:** [2-cross-region-replication.md](./2-cross-region-replication.md) — CRR và chiến lược DR
