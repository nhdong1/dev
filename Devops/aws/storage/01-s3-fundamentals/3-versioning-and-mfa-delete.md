# Versioning và MFA Delete — Quản Lý Phiên Bản và Xóa An Toàn

> Versioning (Quản lý phiên bản) bảo vệ dữ liệu khỏi xóa nhầm và ghi đè — MFA Delete (Xóa có xác thực đa yếu tố) thêm lớp bảo vệ chống xóa vô tình hoặc có chủ đích.

---

## 1. Versioning — Quản Lý Phiên Bản

### Versioning là gì?

Khi bật versioning, S3 **không xóa hoặc ghi đè** object cũ — thay vào đó lưu thêm một phiên bản mới. Mỗi phiên bản có **Version ID** (định danh phiên bản) duy nhất.

```
Không có versioning:
  PUT photo.jpg → overwrite (ghi đè, mất bản cũ)
  DELETE photo.jpg → xóa vĩnh viễn

Có versioning:
  PUT photo.jpg v1 → Version ID: aaa111
  PUT photo.jpg v2 → Version ID: bbb222 (v1 vẫn còn)
  DELETE photo.jpg → Tạo "delete marker" (v1, v2 vẫn còn)
```

### Trạng Thái Bucket Versioning

| Trạng Thái | Ý Nghĩa |
|-----------|--------|
| **Unversioned** (Mặc định) | Không có versioning, không có Version ID |
| **Versioning-enabled** (Đã bật) | Mọi object đều có Version ID |
| **Versioning-suspended** (Tạm dừng) | Object mới không có Version ID (null), các phiên bản cũ vẫn giữ |

> **Không thể tắt hoàn toàn** versioning sau khi đã bật — chỉ có thể **suspend** (tạm dừng).

### Bật Versioning

```bash
# Bật versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Kiểm tra trạng thái
aws s3api get-bucket-versioning --bucket my-bucket
# Kết quả:
# {
#   "Status": "Enabled"
# }

# Tạm dừng versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Suspended
```

---

## 2. Cách Versioning Hoạt Động

### Upload và Ghi Đè

```
BEFORE UPLOAD:
Bucket: my-bucket
  photo.jpg [Version: null / trước khi bật versioning]

UPLOAD v1 (sau khi bật versioning):
  photo.jpg [Version: aaa111] ← Current (hiện tại)
  photo.jpg [Version: null]

UPLOAD v2:
  photo.jpg [Version: bbb222] ← Current
  photo.jpg [Version: aaa111]
  photo.jpg [Version: null]
```

### Xóa Object Với Versioning

```
BEFORE DELETE:
  photo.jpg [Version: bbb222] ← Current
  photo.jpg [Version: aaa111]

DELETE photo.jpg (không chỉ định version):
  photo.jpg [Delete Marker: ccc333] ← Current (marker)
  photo.jpg [Version: bbb222]
  photo.jpg [Version: aaa111]

→ Object "ẩn" — không thấy khi list, nhưng dữ liệu vẫn còn
→ Dùng GetObject sẽ nhận lỗi 404

HARD DELETE — xóa version cụ thể:
  DELETE photo.jpg?versionId=bbb222
  → Xóa vĩnh viễn version bbb222
```

### Khôi Phục Object Đã Xóa

```bash
# Liệt kê tất cả phiên bản
aws s3api list-object-versions --bucket my-bucket --prefix photo.jpg

# Xóa delete marker để "undelete" (khôi phục)
aws s3api delete-object \
  --bucket my-bucket \
  --key photo.jpg \
  --version-id ccc333   # Version ID của delete marker

# Hoặc download phiên bản cũ
aws s3api get-object \
  --bucket my-bucket \
  --key photo.jpg \
  --version-id aaa111 \
  photo_v1_restored.jpg
```

---

## 3. Chi Phí Của Versioning

Mỗi phiên bản lưu trữ tốn phí riêng — tổng chi phí tăng theo số phiên bản:

```
Ví dụ thực tế:
  photo.jpg v1: 5MB
  photo.jpg v2: 5.2MB (chỉnh sửa nhỏ)
  photo.jpg v3: 5.1MB

→ Tổng lưu trữ: 15.3MB (không phải 5.1MB)
→ Chi phí tăng 3x
```

### Kiểm Soát Chi Phí Versioning

**Lifecycle rules** (Quy tắc vòng đời) để tự động dọn phiên bản cũ:

```json
{
  "Rules": [
    {
      "ID": "Delete old versions after 90 days",
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
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

---

## 4. Delete Marker — Đánh Dấu Xóa

**Delete marker** là một object đặc biệt không có dữ liệu — chỉ là nhãn đánh dấu "đã xóa":

| Đặc Điểm | Giá Trị |
|---------|--------|
| Kích thước | 0 byte |
| Dữ liệu | Không có |
| Có thể download? | Không |
| Ảnh hưởng đến LIST | Object bị ẩn |
| Ảnh hưởng đến GET | Trả về 404 |
| Cách xóa | Xóa delete marker theo version ID |

### Phân biệt Soft Delete và Hard Delete

```
Soft Delete (xóa mềm) — DELETE không chỉ định versionId:
  → Tạo delete marker
  → Dữ liệu vẫn còn, có thể khôi phục
  → Ứng dụng thấy 404

Hard Delete (xóa cứng) — DELETE với versionId cụ thể:
  → Xóa vĩnh viễn phiên bản đó
  → Không thể khôi phục
  → Cần quyền s3:DeleteObjectVersion
```

---

## 5. MFA Delete — Xóa Có Xác Thực Đa Yếu Tố

### MFA Delete là gì?

**MFA Delete** yêu cầu xác thực thêm bằng thiết bị **MFA** (Multi-Factor Authentication — Xác thực Đa Yếu Tố — thường là TOTP token) trước khi:

1. **Xóa vĩnh viễn** một phiên bản cụ thể (hard delete)
2. **Thay đổi trạng thái versioning** (Enable ↔ Suspend)

### Yêu Cầu Để Dùng MFA Delete

| Yêu Cầu | Chi Tiết |
|---------|---------|
| Bật versioning | MFA Delete chỉ hoạt động khi versioning đã bật |
| Quyền root account | Chỉ AWS **root user** mới có thể bật/tắt MFA Delete |
| Thiết bị MFA | Virtual MFA (Google Authenticator) hoặc hardware MFA |
| Gọi API | Phải dùng AWS CLI hoặc SDK — Console không hỗ trợ |

### Bật MFA Delete

```bash
# Chỉ root user mới thực hiện được lệnh này
aws s3api put-bucket-versioning \
  --bucket my-critical-bucket \
  --versioning-configuration '{
    "Status": "Enabled",
    "MFADelete": "Enabled"
  }' \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa 123456"
  #      └─── ARN thiết bị MFA ───────────────────────┘ └──── Mã TOTP ──┘
```

### Xóa Object Khi MFA Delete Được Bật

```bash
# Cần cung cấp mã MFA khi xóa version cụ thể
aws s3api delete-object \
  --bucket my-critical-bucket \
  --key sensitive-file.pdf \
  --version-id aaa111 \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa 654321"
```

### Khi Nào Cần MFA Delete?

```
Dùng MFA Delete khi:
✅ Bucket chứa dữ liệu không thể khôi phục (financial records, medical data)
✅ Môi trường tuân thủ pháp lý nghiêm ngặt (compliance-regulated environments)
✅ Muốn bảo vệ khỏi tấn công ransomware xóa backup
✅ Dữ liệu production quan trọng

Không cần MFA Delete khi:
❌ Bucket test/development
❌ Dữ liệu có thể tái tạo
❌ Lifecycle rules tự động dọn — MFA Delete can thiệp lifecycle
```

---

## 6. Object Lock — Khóa Đối Tượng (So Sánh Với MFA Delete)

| Tính Năng | MFA Delete | Object Lock |
|-----------|-----------|------------|
| **Mục đích** | Bảo vệ xóa bằng xác thực | Ngăn chặn xóa trong khoảng thời gian |
| **Cơ chế** | Yêu cầu MFA token | WORM — Write Once Read Many |
| **Khôi phục** | Có thể xóa nếu có MFA | Không thể xóa trong retention period |
| **Dùng cho** | Bảo vệ thông thường | Compliance nghiêm ngặt (SEC, FINRA) |
| **Bật khi nào** | Sau khi tạo bucket | **Phải bật lúc tạo bucket** |

---

## 7. Best Practices (Thực Hành Tốt Nhất)

### Cấu Hình Recommended cho Bucket Production

```bash
# 1. Bật versioning
aws s3api put-bucket-versioning \
  --bucket prod-bucket \
  --versioning-configuration Status=Enabled

# 2. Bật MFA Delete (dùng root account)
aws s3api put-bucket-versioning \
  --bucket prod-bucket \
  --versioning-configuration '{
    "Status": "Enabled",
    "MFADelete": "Enabled"
  }' \
  --mfa "arn:aws:iam::ACCOUNT-ID:mfa/root-account-mfa TOKEN"

# 3. Thêm lifecycle rule để giới hạn chi phí versioning
aws s3api put-bucket-lifecycle-configuration \
  --bucket prod-bucket \
  --lifecycle-configuration file://lifecycle.json
```

### Giám Sát Versioning

```bash
# Kiểm tra tổng số phiên bản (có thể tốn kém nếu nhiều)
aws s3api list-object-versions \
  --bucket my-bucket \
  --query 'length(Versions)' \
  --output text

# Tìm delete markers
aws s3api list-object-versions \
  --bucket my-bucket \
  --query 'DeleteMarkers[*].{Key:Key,VersionId:VersionId}'
```

---

## Câu Hỏi Phỏng Vấn

**Q: Versioning và Replication liên quan nhau như thế nào?**
> Replication (CRR/SRR) **yêu cầu versioning** phải được bật trên cả source và destination bucket. Khi replicate, các phiên bản cũng được sao chép nếu cấu hình đúng.

**Q: Xóa delete marker có tốn phí không?**
> Không. Xóa delete marker không tốn phí retrieval. Nhưng tạo và lưu delete marker có tốn phí lưu trữ nhỏ (vì marker có size 0 nhưng AWS vẫn tính minimum).

**Q: Có thể dùng lifecycle rules với MFA Delete không?**
> Lifecycle rules tự động xóa các phiên bản cũ — hoạt động bình thường ngay cả khi MFA Delete được bật. MFA Delete chỉ áp dụng cho thao tác **thủ công** qua API/CLI/Console.

**Q: Nếu bucket có 1 triệu object, mỗi object có 5 phiên bản, chi phí tăng thế nào?**
> Tăng gấp ~5 lần cho storage cost. Phải dùng lifecycle rules `NoncurrentVersionExpiration` để tự động xóa phiên bản cũ. AWS khuyến nghị giới hạn không quá 100,000 phiên bản mỗi object.

**Q: Tại sao chỉ root user mới bật được MFA Delete?**
> Để ngăn IAM admin bị compromise xóa MFA Delete và sau đó xóa dữ liệu. Root user là tầng bảo vệ cuối cùng — không thể bị IAM policy kiểm soát.

---

## Điều Hướng

- **Tiếp theo:** [4-presigned-urls.md](./4-presigned-urls.md) — Presigned URLs
- **Quay lại:** [2-storage-classes.md](./2-storage-classes.md) — Storage Classes
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
