# Thiết Kế Lifecycle Rules — Ví Dụ Thực Tế

> Lifecycle Rules — Quy Tắc Vòng Đời S3 tự động chuyển object giữa các storage class hoặc xóa object khi hết hạn, giúp giảm chi phí lưu trữ mà không cần can thiệp thủ công.

## 📚 Mục Lục

1. [Tổng Quan Lifecycle Rules](#1-tổng-quan-lifecycle-rules)
2. [Các Hành Động Lifecycle](#2-các-hành-động-lifecycle)
3. [Thiết Kế Lifecycle Cho Log Storage](#3-thiết-kế-lifecycle-cho-log-storage)
4. [Thiết Kế Lifecycle Cho Media/Content](#4-thiết-kế-lifecycle-cho-mediacontent)
5. [Thiết Kế Lifecycle Cho Backup](#5-thiết-kế-lifecycle-cho-backup)
6. [Xử Lý Versioned Objects — Object Có Phiên Bản](#6-xử-lý-versioned-objects)
7. [Cleanup Incomplete Multipart Uploads](#7-cleanup-incomplete-multipart-uploads)
8. [Lifecycle Rules Theo AWS CLI](#8-lifecycle-rules-theo-aws-cli)
9. [Pitfalls — Cạm Bẫy Thường Gặp](#9-pitfalls)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Lifecycle Rules

### Lifecycle Rules Là Gì

```
Lifecycle Rule = Điều Kiện (Filter) + Hành Động (Action) + Thời Gian (When)

Ví dụ đơn giản:
  Điều kiện: Tất cả object có prefix "logs/"
  Hành động: Chuyển sang Glacier
  Thời gian: Sau 90 ngày kể từ khi tạo

Kết quả tự động:
  Ngày 1: Object tạo → Standard ($0.023/GB)
  Ngày 90: S3 tự động chuyển → Glacier ($0.0036/GB)
  → Tiết kiệm 84% chi phí lưu trữ sau ngày 90
```

### Cách Lifecycle Rules Hoạt Động

```
Timeline của một object:

Ngày 0: Object created (tạo) → Standard
         │
         ▼ (Lifecycle rule: 30 ngày)
Ngày 30: Transition (chuyển) → Standard-IA
         │
         ▼ (Lifecycle rule: thêm 60 ngày)
Ngày 90: Transition → Glacier Flexible
         │
         ▼ (Lifecycle rule: thêm 2,555 ngày)
Ngày 2,645: Expiration (xóa) — tổng 7.25 năm

Toàn bộ tự động, không can thiệp thủ công!
```

### Giới Hạn Cần Biết

```
- Mỗi bucket: Tối đa 1,000 lifecycle rules
- Thời gian chuyển tối thiểu:
  Standard → Standard-IA: Tối thiểu 30 ngày sau khi tạo
  Standard → Glacier: Tối thiểu 1 ngày
  Standard-IA → Glacier: Tối thiểu 30 ngày ở Standard-IA
  Glacier Flexible → Deep Archive: Tối thiểu 90 ngày ở Glacier Flexible
- Filter có thể theo: prefix, tag, object size, hoặc kết hợp
```

---

## 2. Các Hành Động Lifecycle

### Nhóm Hành Động

```
1. TRANSITION (Chuyển Tầng):
   Standard → Standard-IA
   Standard → One Zone-IA
   Standard → Glacier Instant Retrieval
   Standard → Glacier Flexible Retrieval
   Standard → Glacier Deep Archive
   Standard-IA → Glacier Flexible
   ...

2. EXPIRATION (Xóa):
   - Xóa current version (phiên bản hiện tại)
   - Xóa noncurrent version (phiên bản cũ khi versioning bật)
   - Xóa delete markers (đánh dấu xóa) hết hạn

3. ABORT INCOMPLETE MULTIPART UPLOAD (Hủy Upload Dang Dở):
   - Tự động hủy multipart upload không hoàn thành
```

### Sơ Đồ Chuyển Tầng Hợp Lệ

```
                    Standard
                       │
          ┌────────────┼────────────┬──────────────┐
          ▼            ▼            ▼              ▼
    Standard-IA   One Zone-IA  Glacier Instant  Glacier Flexible
          │            │                            │
          │            │                            ▼
          └────────────┴────────────────────► Deep Archive
          
Lưu ý:
  ✅ Có thể chuyển xuống (cheaper classes)
  ❌ KHÔNG thể chuyển ngược lên (phải copy thủ công)
  ✅ Có thể skip tầng (Standard → Glacier thẳng, bỏ qua IA)
```

---

## 3. Thiết Kế Lifecycle Cho Log Storage

### Kịch Bản: Application Logs — Log Ứng Dụng

```
Yêu cầu nghiệp vụ:
  - Log 7 ngày gần: Cần truy cập nhanh để debug
  - Log 7–90 ngày: Truy cập thỉnh thoảng (điều tra sự cố)
  - Log 90 ngày – 7 năm: Lưu cho compliance, hiếm truy cập
  - Sau 7 năm: Xóa

Access Pattern (Mẫu Truy Cập):
  Hàng ngày         → Standard (truy cập thường xuyên)
  Tuần 2–13         → Standard-IA (truy cập thỉnh thoảng)
  Tháng 3–84        → Glacier Flexible (hiếm, compliance)
  Sau 84 tháng      → Expiration (xóa)
```

### Lifecycle Configuration — Cấu Hình Vòng Đời

```json
{
  "Rules": [
    {
      "ID": "log-lifecycle-7-years",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "application-logs/"
      },
      "Transitions": [
        {
          "Days": 7,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### Tính Toán Tiết Kiệm

```
Dữ liệu log: 10 GB/ngày
Sau 1 năm ổn định:

KHÔNG có lifecycle (tất cả Standard):
  365 × 10 = 3,650 GB × $0.023 = $83.95/tháng

CÓ lifecycle rule:
  Standard (7 ngày): 70 GB × $0.023 = $1.61
  Standard-IA (83 ngày): 830 GB × $0.0125 = $10.38
  Glacier Flexible (275 ngày): 2,750 GB × $0.0036 = $9.90
  Tổng: $21.89/tháng

Tiết kiệm: $62.06/tháng = 74%!
Sau 1 năm tiết kiệm: ~$745
```

---

## 4. Thiết Kế Lifecycle Cho Media/Content

### Kịch Bản: Video Platform — Nền Tảng Video

```
Yêu cầu:
  - Video mới upload (1 tháng đầu): Xem nhiều → Standard
  - Video 1–6 tháng: Xem vừa phải → Standard-IA
  - Video >6 tháng: Ít xem → Glacier Instant (để vẫn xem được nhanh)
  - Video >3 năm: Lưu trữ dài hạn → Glacier Flexible
  - Video xóa bởi user: Xóa ngay

Lưu ý: Glacier Instant (không phải Flexible) vì người dùng vẫn cần xem video,
chỉ là ít thường xuyên hơn — không thể chờ 3–5 giờ retrieve.
```

```json
{
  "Rules": [
    {
      "ID": "video-content-lifecycle",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "videos/",
          "Tags": [
            {"Key": "type", "Value": "user-upload"}
          ]
        }
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 180,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 1095,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "ID": "video-thumbnail-lifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "thumbnails/"
      },
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        }
      ]
    }
  ]
}
```

### Kịch Bản: E-Commerce Product Images — Ảnh Sản Phẩm

```
Đặc thù:
  - Ảnh sản phẩm đang bán: Truy cập thường xuyên → Standard
  - Ảnh sản phẩm ngừng bán (discontinued): Ít truy cập → Standard-IA
  - Ảnh >5 năm không truy cập: Archive → Glacier

Cách triển khai: Tag object khi sản phẩm ngừng bán
  Tag: "status" = "discontinued"
  Lifecycle: Object có tag này → Standard-IA sau 30 ngày
```

```json
{
  "Rules": [
    {
      "ID": "discontinued-product-images",
      "Status": "Enabled",
      "Filter": {
        "Tag": {"Key": "status", "Value": "discontinued"}
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 1825,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 3650
      }
    }
  ]
}
```

---

## 5. Thiết Kế Lifecycle Cho Backup

### Kịch Bản: Database Backup — Sao Lưu Cơ Sở Dữ Liệu

```
Yêu cầu:
  - Backup hàng ngày, giữ 30 ngày gần nhất
  - Backup cuối tuần, giữ 3 tháng
  - Backup cuối tháng, giữ 1 năm
  - Backup cuối năm, giữ 7 năm (compliance)

Cấu trúc prefix:
  backups/daily/YYYY-MM-DD/
  backups/weekly/YYYY-WW/
  backups/monthly/YYYY-MM/
  backups/yearly/YYYY/
```

```json
{
  "Rules": [
    {
      "ID": "daily-backup-30d",
      "Status": "Enabled",
      "Filter": {"Prefix": "backups/daily/"},
      "Expiration": {"Days": 30}
    },
    {
      "ID": "weekly-backup-13-weeks",
      "Status": "Enabled",
      "Filter": {"Prefix": "backups/weekly/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"}
      ],
      "Expiration": {"Days": 91}
    },
    {
      "ID": "monthly-backup-1-year",
      "Status": "Enabled",
      "Filter": {"Prefix": "backups/monthly/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365}
    },
    {
      "ID": "yearly-backup-7-years",
      "Status": "Enabled",
      "Filter": {"Prefix": "backups/yearly/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }
  ]
}
```

---

## 6. Xử Lý Versioned Objects

### Versioning Lifecycle — Vòng Đời Với Phiên Bản

```
Khi Versioning bật, mỗi PUT tạo một phiên bản mới:
  Version 1 (oldest) → NoncurrentVersion (phiên bản không hiện tại)
  Version 2          → NoncurrentVersion
  Version 3 (latest) → Current Version (phiên bản hiện tại)

Lifecycle cần quản lý CẢ HAI loại:
  - Current version: Transition như bình thường
  - Noncurrent version: Cần xóa riêng, không tự xóa theo current
```

```json
{
  "Rules": [
    {
      "ID": "manage-versioned-objects",
      "Status": "Enabled",
      "Filter": {},
      "Transitions": [
        {"Days": 90, "StorageClass": "STANDARD_IA"}
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "NoncurrentDays": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 365,
        "NewerNoncurrentVersions": 3
      }
    }
  ]
}
```

```
NoncurrentVersionExpiration settings — Cài đặt xóa phiên bản cũ:
  NoncurrentDays: 365 → Xóa version cũ sau 365 ngày kể từ lúc bị thay thế
  NewerNoncurrentVersions: 3 → Luôn giữ 3 version cũ nhất, xóa cái cũ hơn

Ví dụ: File được PUT 10 lần trong 1 năm:
  Current: Version 10
  Noncurrent: Versions 9, 8, 7 (giữ 3 cái mới nhất)
  Versions 1–6: Xóa nếu >365 ngày kể từ bị thay thế
```

### Xóa Delete Markers Hết Hạn

```
Delete Marker (Đánh Dấu Xóa) tạo ra khi:
  - Bucket có versioning
  - User DELETE object
  - S3 không xóa thật mà tạo delete marker

Vấn đề: Delete markers cũ tích lũy, tốn dung lượng nhỏ nhưng ảnh hưởng LIST

Giải pháp:
```

```json
{
  "Rules": [
    {
      "ID": "cleanup-delete-markers",
      "Status": "Enabled",
      "Filter": {},
      "Expiration": {
        "ExpiredObjectDeleteMarker": true
      }
    }
  ]
}
```

---

## 7. Cleanup Incomplete Multipart Uploads

### Tại Sao Quan Trọng

```
Multipart Upload — Upload Nhiều Phần:
  - Upload file lớn được chia thành nhiều parts
  - Nếu upload bị interrupted (gián đoạn), parts đã upload không tự xóa
  - Mỗi part tính phí như object thông thường
  - Có thể tích lũy GBs dữ liệu "rác" trong vài tháng

Kiểm tra incomplete uploads:
```

```bash
# Liệt kê tất cả multipart uploads đang dở
aws s3api list-multipart-uploads --bucket my-bucket

# Kiểm tra dung lượng parts của một upload
aws s3api list-parts \
  --bucket my-bucket \
  --key my-large-file.zip \
  --upload-id "upload_id_here"
```

### Lifecycle Rule Cleanup

```json
{
  "Rules": [
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

```
Giải thích: Bất kỳ multipart upload nào không hoàn thành sau 7 ngày
sẽ tự động bị abort (hủy) và tất cả parts bị xóa.

Nên thêm rule này vào MỌI bucket — không có tác dụng phụ,
chỉ dọn dẹp dữ liệu rác.
```

---

## 8. Lifecycle Rules Theo AWS CLI

### Cách Áp Dụng Lifecycle Policy

```bash
# Tạo file lifecycle.json
cat > lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "log-archive-policy",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 2555},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
EOF

# Áp dụng lifecycle policy
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-logs-bucket \
  --lifecycle-configuration file://lifecycle.json

# Xem lifecycle policy hiện tại
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-logs-bucket

# Xóa lifecycle policy
aws s3api delete-bucket-lifecycle --bucket my-logs-bucket
```

### Kiểm Tra Lifecycle Đang Hoạt Động

```bash
# Xem storage class hiện tại của object
aws s3api head-object \
  --bucket my-logs-bucket \
  --key logs/2025-01-01/app.log

# Output bao gồm "StorageClass" cho biết class hiện tại

# Liệt kê object với storage class (để verify transition)
aws s3api list-objects-v2 \
  --bucket my-logs-bucket \
  --prefix logs/ \
  --query 'Contents[*].{Key:Key,StorageClass:StorageClass,Size:Size}' \
  --output table
```

---

## 9. Pitfalls

### Lỗi Thường Gặp Khi Thiết Kế Lifecycle

```
❌ Sai: Chuyển object <128KB sang Standard-IA
   → Bị tính 128KB minimum, chi phí không giảm
   Đúng: Filter theo object size minimum 128KB
   
   "Filter": {
     "And": {
       "Prefix": "logs/",
       "ObjectSizeGreaterThan": 131072
     }
   }

❌ Sai: Lifecycle chuyển Standard → Standard-IA sau 5 ngày
   → Minimum là 30 ngày. AWS sẽ báo lỗi.

❌ Sai: Transition Standard-IA → Standard (đi ngược lên)
   → Không hỗ trợ. Phải copy object thủ công để đi lên class cao hơn.

❌ Sai: Không có rule cho noncurrent versions khi dùng versioning
   → Versions cũ tích lũy không giới hạn, chi phí tăng không kiểm soát.

❌ Sai: Quên rule AbortIncompleteMultipartUpload
   → Parts upload dở tích lũy theo thời gian, tốn tiền vô ích.

❌ Sai: Dùng Glacier Flexible cho data cần truy cập nhanh
   → Retrieve mất 3–5 giờ, phù hợp với cold archive không phải data hoạt động.
```

### Cách Test Lifecycle Rule Trước Khi Áp Dụng

```bash
# Tạo bucket test nhỏ
aws s3 mb s3://lifecycle-test-$(date +%s)

# Upload object test
aws s3 cp test.log s3://lifecycle-test-xxx/logs/test.log

# Áp dụng lifecycle với ngày ngắn để test
# (Lifecycle chạy hàng ngày, không chạy ngay lập tức)

# Dùng S3 Inventory để xem storage class của object
# (Thay vì chờ lifecycle chạy thật)
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Tối thiểu bao nhiêu ngày object cần ở Standard trước khi chuyển sang Standard-IA?**

> Tối thiểu 30 ngày. Đây là giới hạn của S3 lifecycle — không thể cấu hình chuyển sang Standard-IA (hoặc One Zone-IA) trước 30 ngày kể từ khi object được tạo. Tuy nhiên, có thể chuyển sang Glacier trực tiếp từ Standard sau chỉ 1 ngày, bỏ qua IA hoàn toàn.

**Q: Khi versioning bật, lifecycle expiration hoạt động như thế nào?**

> Khi versioning bật, lifecycle expiration trên current version không xóa object vĩnh viễn mà tạo delete marker, biến current version thành noncurrent. Để xóa thật sự, cần cấu hình riêng `NoncurrentVersionExpiration` cho noncurrent versions và `ExpiredObjectDeleteMarker: true` để dọn delete markers. Thiếu hai rule này, versioned objects sẽ tích lũy vô hạn.

**Q: Tại sao nên luôn có AbortIncompleteMultipartUpload trong lifecycle?**

> Upload nhiều phần (multipart) bị gián đoạn để lại các parts chưa hoàn thành vẫn chiếm dung lượng và bị tính phí như object bình thường, nhưng không thể truy cập hay dùng. Theo thời gian, đây có thể là GBs lãng phí. Rule AbortIncompleteMultipartUpload tự động dọn sạch sau số ngày định sẵn (thường 7 ngày) mà không có tác dụng phụ gì.

**Q: Thiết kế lifecycle cho 100 TB log, yêu cầu giữ 7 năm nhưng chỉ truy cập thường xuyên trong 30 ngày đầu?**

> Thiết kế ba tầng: (1) 0–30 ngày: Standard — truy cập nhanh debug hàng ngày; (2) 30–90 ngày: Standard-IA — truy cập thỉnh thoảng khi điều tra, phí retrieve chấp nhận được; (3) 90 ngày–7 năm: Glacier Flexible — compliance archive, retrieve trong vài giờ là đủ; (4) Ngày 2,555: Expiration. Với 100 TB log tích lũy, tiết kiệm so với để toàn Standard khoảng 70–80% chi phí hàng tháng.

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
