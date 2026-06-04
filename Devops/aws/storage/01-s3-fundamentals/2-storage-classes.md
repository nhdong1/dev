# Storage Classes — Lớp Lưu Trữ S3

> S3 cung cấp 7 storage classes (lớp lưu trữ) với các mức chi phí, hiệu suất và độ bền khác nhau — chọn đúng class giúp tiết kiệm chi phí đáng kể.

---

## Tổng Quan Các Storage Class

```
Chi phí lưu trữ (cao → thấp)
────────────────────────────────────────────────────────
Standard          $$$   Truy cập thường xuyên
Standard-IA        $$   Truy cập không thường xuyên
Intelligent-Tiering $$  Tự động chọn tầng
One Zone-IA        $    Không thường xuyên, 1 AZ
Glacier Instant    $    Lưu trữ lạnh, lấy ngay
Glacier Flexible   ¢¢  Lưu trữ lạnh, 1–12 giờ
Glacier Deep Archive ¢  Lưu trữ lạnh nhất, 12–48 giờ
────────────────────────────────────────────────────────
```

---

## 1. S3 Standard — Tiêu Chuẩn

**Dành cho:** Dữ liệu truy cập thường xuyên (nhiều lần mỗi tháng hoặc mỗi ngày)

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** (Độ bền) | 99.999999999% (11 chín) |
| **Availability** (Tính sẵn sàng) | 99.99% |
| **Số AZ lưu trữ** | ≥ 3 |
| **Min storage duration** (Thời gian lưu tối thiểu) | Không có |
| **Min object size** | Không có |
| **Retrieval fee** (Phí lấy dữ liệu) | Không có |
| **Giá lưu trữ** | ~$0.023/GB/tháng (ap-southeast-1) |

**Use cases (Trường hợp sử dụng):**
- Web content (hình ảnh, CSS, JS)
- Big data analytics — dữ liệu đang xử lý
- Mobile và gaming apps
- Content distribution (phân phối nội dung)

```bash
# Upload với Standard class (mặc định)
aws s3 cp photo.jpg s3://my-bucket/photo.jpg
# hoặc chỉ định rõ
aws s3 cp photo.jpg s3://my-bucket/photo.jpg --storage-class STANDARD
```

---

## 2. S3 Standard-IA — Infrequent Access (Truy Cập Không Thường Xuyên)

**Dành cho:** Dữ liệu cần giữ lâu dài nhưng ít khi truy cập (dưới 1 lần/tháng)

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% |
| **Availability** | 99.9% |
| **Số AZ lưu trữ** | ≥ 3 |
| **Min storage duration** | **30 ngày** |
| **Min billable object size** | **128 KB** |
| **Retrieval fee** | Có — tính theo GB lấy ra |
| **Giá lưu trữ** | ~$0.0125/GB/tháng |

**Use cases:**
- Backup dài hạn (DR, disaster recovery)
- Log files sau 30 ngày
- Dữ liệu tuân thủ (compliance data) ít truy cập
- Disaster recovery files

> **Cạm bẫy chi phí:** Nếu xóa object trước 30 ngày → vẫn bị tính phí 30 ngày. Object < 128KB → tính phí như 128KB.

```bash
aws s3 cp backup.tar.gz s3://my-bucket/backups/ --storage-class STANDARD_IA
```

---

## 3. S3 One Zone-IA — Truy Cập Không Thường Xuyên Một Vùng

**Dành cho:** Dữ liệu ít truy cập, có thể tạo lại nếu AZ bị lỗi

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% (trong 1 AZ) |
| **Availability** | 99.5% |
| **Số AZ lưu trữ** | **1 AZ duy nhất** |
| **Min storage duration** | **30 ngày** |
| **Min billable size** | **128 KB** |
| **Giá lưu trữ** | ~$0.01/GB/tháng (rẻ hơn 20% so với Standard-IA) |

**Use cases:**
- Secondary backup copies (bản sao backup phụ)
- Thumbnail images (có thể tạo lại từ ảnh gốc)
- Dữ liệu có thể tái tạo dễ dàng

> **Rủi ro:** Nếu AZ chứa dữ liệu bị mất (thiên tai, lỗi nghiêm trọng) → **mất dữ liệu vĩnh viễn**.

```bash
aws s3 cp thumbnail.jpg s3://my-bucket/thumbs/ --storage-class ONEZONE_IA
```

---

## 4. S3 Intelligent-Tiering — Phân Tầng Thông Minh

**Dành cho:** Dữ liệu có access pattern không dự đoán được

### Cách hoạt động

S3 tự động di chuyển object giữa các tầng dựa trên tần suất truy cập:

```
Tầng Frequent Access (truy cập thường xuyên)
    ↕ Sau 30 ngày không truy cập
Tầng Infrequent Access (truy cập không thường xuyên)
    ↕ Sau 90 ngày không truy cập (tùy chọn, kích hoạt riêng)
Tầng Archive Instant Access (lưu trữ lấy ngay)
    ↕ Sau 90–180+ ngày (kích hoạt riêng)
Tầng Archive Access (lưu trữ, lấy trong vài giờ)
    ↕ Sau 180+ ngày (kích hoạt riêng)
Tầng Deep Archive Access (lưu trữ sâu nhất)
```

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% |
| **Availability** | 99.9% |
| **Min storage duration** | **30 ngày** |
| **Min object size** | **128 KB** (nhỏ hơn → tính phí Standard) |
| **Monitoring fee** (Phí giám sát) | $0.0025/1,000 objects/tháng |
| **Retrieval fee** | Không có (cho 2 tầng đầu) |

**Use cases:**
- Data lake với access pattern biến đổi
- User-generated content (nội dung người dùng tạo)
- Log archives với truy cập không đều
- Backup mà đôi khi cần phân tích

```bash
aws s3 cp data.parquet s3://my-bucket/lake/ \
  --storage-class INTELLIGENT_TIERING
```

---

## 5. S3 Glacier Instant Retrieval — Glacier Lấy Tức Thì

**Dành cho:** Archive (lưu trữ) cần lấy ngay trong vài mili giây

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% |
| **Availability** | 99.9% |
| **Thời gian lấy dữ liệu** | **Mili giây** |
| **Min storage duration** | **90 ngày** |
| **Min billable size** | **128 KB** |
| **Giá lưu trữ** | ~$0.004/GB/tháng |

**Use cases:**
- Medical images (ảnh y tế) ít truy cập nhưng cần ngay khi cần
- News media archive (lưu trữ tin tức)
- Dữ liệu truy cập theo quý

```bash
aws s3 cp medical-scan.dcm s3://my-bucket/archive/ \
  --storage-class GLACIER_IR
```

---

## 6. S3 Glacier Flexible Retrieval — Glacier Linh Hoạt

**Dành cho:** Archive dài hạn, không cần lấy ngay

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% |
| **Thời gian lấy dữ liệu** | **1–5 phút** (Expedited), **3–5 giờ** (Standard), **5–12 giờ** (Bulk) |
| **Min storage duration** | **90 ngày** |
| **Giá lưu trữ** | ~$0.0036/GB/tháng |
| **Phí lấy dữ liệu** | Tùy theo tier (Expedited > Standard > Bulk) |

### Retrieval tiers (Tầng lấy dữ liệu)

| Tier | Thời Gian | Chi Phí | Dùng Khi |
|------|----------|--------|---------|
| **Expedited** (Khẩn cấp) | 1–5 phút | Cao nhất | Cần ngay, trường hợp khẩn |
| **Standard** (Tiêu chuẩn) | 3–5 giờ | Vừa | Bình thường |
| **Bulk** (Hàng loạt) | 5–12 giờ | Thấp nhất | Lấy số lượng lớn |

**Use cases:**
- Compliance archives (lưu trữ tuân thủ pháp lý)
- Backup lâu dài 5–10 năm
- Tape replacement (thay thế băng từ vật lý)

```bash
aws s3 cp compliance-data.zip s3://my-bucket/compliance/ \
  --storage-class GLACIER

# Khởi tạo restore (phải restore trước khi download)
aws s3api restore-object \
  --bucket my-bucket \
  --key compliance/compliance-data.zip \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}'
```

---

## 7. S3 Glacier Deep Archive — Lưu Trữ Sâu

**Dành cho:** Lưu trữ dài hạn nhất, ít khi cần lấy, chi phí thấp nhất

| Thuộc Tính | Giá Trị |
|-----------|--------|
| **Durability** | 99.999999999% |
| **Thời gian lấy dữ liệu** | **12 giờ** (Standard), **48 giờ** (Bulk) |
| **Min storage duration** | **180 ngày** |
| **Giá lưu trữ** | ~$0.00099/GB/tháng |

**Use cases:**
- Lưu trữ 7–10 năm theo quy định pháp lý
- Digital preservation (bảo tồn kỹ thuật số)
- Healthcare, tài chính, tuân thủ pháp lý dài hạn

```bash
aws s3 cp annual-report-2016.pdf s3://my-bucket/archive/ \
  --storage-class DEEP_ARCHIVE
```

---

## Bảng So Sánh Đầy Đủ

| | Standard | Standard-IA | One Zone-IA | Intelligent-Tiering | Glacier Instant | Glacier Flexible | Glacier Deep |
|--|---------|------------|------------|--------------------|-----------------|--------------------|-------------|
| **Durability** | 11 chín | 11 chín | 11 chín (1 AZ) | 11 chín | 11 chín | 11 chín | 11 chín |
| **Availability** | 99.99% | 99.9% | 99.5% | 99.9% | 99.9% | N/A | N/A |
| **AZ** | ≥3 | ≥3 | 1 | ≥3 | ≥3 | ≥3 | ≥3 |
| **Min duration** | Không | 30 ngày | 30 ngày | 30 ngày | 90 ngày | 90 ngày | 180 ngày |
| **Retrieval time** | Ms | Ms | Ms | Ms | Ms | 1–12 giờ | 12–48 giờ |
| **Retrieval fee** | Không | Có | Có | Không* | Có | Có | Có |
| **Giá/GB** | $0.023 | $0.0125 | $0.010 | Thay đổi | $0.004 | $0.0036 | $0.001 |

*Intelligent-Tiering không có retrieval fee ở 2 tầng đầu.

---

## Quy Tắc Chọn Storage Class

```
Truy cập thường xuyên (hàng ngày)?
├── Có → Standard
└── Không → Tiếp tục...
    │
    Biết trước access pattern?
    ├── Không → Intelligent-Tiering
    └── Có → Tiếp tục...
        │
        Cần tính sẵn sàng cao?
        ├── Có → Standard-IA
        └── Có thể mất nếu AZ chết? → One Zone-IA
            │
            Lưu trữ lâu, cần lấy ngay?
            ├── Có → Glacier Instant
            └── Không →
                │
                Chấp nhận chờ vài giờ?
                ├── Có (và < 180 ngày) → Glacier Flexible
                └── Lưu > 180 ngày, chấp nhận 12–48h → Glacier Deep Archive
```

---

## Lifecycle Policies — Tự Động Chuyển Tầng

Thay vì chọn class thủ công, dùng **lifecycle rules** để tự động:

```json
{
  "Rules": [
    {
      "ID": "Move to IA after 30 days, then Glacier after 90",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng Intelligent-Tiering thay vì Standard-IA?**
> Dùng Intelligent-Tiering khi không dự đoán được access pattern. Standard-IA tốt hơn khi biết chắc dữ liệu ít truy cập (ít hơn 1 lần/tháng) và kích thước object đủ lớn (>128KB). Intelligent-Tiering có monitoring fee $0.0025/1000 objects — với nhiều object nhỏ chi phí này có thể cao hơn Standard.

**Q: Tại sao Standard-IA có minimum storage duration 30 ngày?**
> AWS tính chi phí tối thiểu 30 ngày — nếu bạn xóa object sau 10 ngày, vẫn bị tính phí 30 ngày. Điều này phản ánh cách AWS thu hồi chi phí cho dữ liệu infrequent access.

**Q: Glacier Flexible Retrieval khác Glacier Instant như thế nào?**
> Glacier Instant: lấy dữ liệu trong mili giây (như Standard), chi phí lưu trữ thấp hơn, nhưng có retrieval fee. Glacier Flexible: lấy trong 1–12 giờ, rẻ hơn, dùng cho archive thực sự không cần ngay.

**Q: Có thể kết hợp nhiều storage class trong cùng một bucket không?**
> Có. Mỗi object có thể có storage class riêng. Lifecycle rules tự động chuyển tầng theo thời gian. Đây là cách phổ biến nhất để tối ưu chi phí.

---

## Điều Hướng

- **Tiếp theo:** [3-versioning-and-mfa-delete.md](./3-versioning-and-mfa-delete.md) — Versioning và MFA Delete
- **Quay lại:** [1-bucket-and-object-model.md](./1-bucket-and-object-model.md) — Bucket và Object Model
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
