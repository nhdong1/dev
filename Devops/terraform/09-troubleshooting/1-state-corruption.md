# State Corruption — Lỗi State File — Phát Hiện và Phục Hồi

> State file — file lưu trạng thái hạ tầng — là thành phần quan trọng nhất trong Terraform. Khi nó bị hỏng, toàn bộ hạ tầng có thể mất khả năng quản lý.

---

## State File Là Gì và Tại Sao Dễ Bị Hỏng?

`terraform.tfstate` là file JSON lưu ánh xạ giữa:
- Resource định nghĩa trong code HCL
- Tài nguyên thực tế đang chạy trên cloud

**Các nguyên nhân gây corrupt — hỏng — state:**

| Nguyên Nhân                              | Mô Tả                                                          |
| ---------------------------------------- | -------------------------------------------------------------- |
| Hai người apply cùng lúc                 | Khi không có state locking — khoá trạng thái                   |
| Interrupt — ngắt — giữa chừng apply      | Ctrl+C, mất kết nối, pod bị kill khi đang apply               |
| Tay xóa resource trực tiếp trên console | Tài nguyên thực mất nhưng state vẫn nghĩ nó tồn tại           |
| Migrate backend không đúng               | Copy state thủ công sai format hoặc thiếu dữ liệu             |
| Lỗi đĩa hoặc object storage             | S3 — Amazon Simple Storage Service — bucket bị xóa nhầm       |
| Chỉnh sửa state file bằng tay           | Sai format JSON hoặc missing required fields                   |

---

## Phát Hiện State Corruption

### Dấu Hiệu Cảnh Báo

```bash
# Lỗi 1: State không parse được
Error: Failed to read state: unexpected end of JSON input

# Lỗi 2: Resource tồn tại trong state nhưng không còn trên cloud
Error refreshing state: "aws_instance.web": instance not found

# Lỗi 3: State version không tương thích
Error: State migration failed: state file version not supported

# Lỗi 4: Checksum mismatch — tổng kiểm tra không khớp
Error: state serial mismatch
```

### Kiểm Tra Thủ Công

```bash
# Bước 1: Tải state xuống
terraform state pull > current-state.json

# Bước 2: Kiểm tra JSON hợp lệ
cat current-state.json | jq . > /dev/null && echo "JSON hợp lệ" || echo "JSON bị lỗi"

# Bước 3: Xem danh sách resource
terraform state list

# Bước 4: Kiểm tra từng resource quan trọng
terraform state show aws_instance.web
terraform state show aws_s3_bucket.data
```

---

## Phân Loại Mức Độ Hỏng

### Mức 1 — Nhẹ: Resource Drift — Lệch Trạng Thái

State tin tài nguyên tồn tại nhưng trên cloud thì không (hoặc ngược lại).

```
State: aws_instance.web (i-abc123) ← tin là tồn tại
Cloud: instance i-abc123 đã bị xóa thủ công

Kết quả: plan báo "Error refreshing state"
```

### Mức 2 — Trung Bình: Partial Apply — Apply Dở Dang

Apply bị interrupt khi đang tạo/cập nhật resource.

```
State ghi nhận:
- Resource A: created ✅
- Resource B: creating... ← bị gián đoạn ở đây
- Resource C: pending (chưa tạo)

Kết quả: state không nhất quán với thực tế
```

### Mức 3 — Nghiêm Trọng: JSON Corrupt — File JSON Hỏng

State file không thể đọc được.

```
State: { "version": 4, "terraform_version": "1.5.0", "serial": 42, "lineage...
↑ File bị cắt giữa chừng, JSON không hoàn chỉnh
```

### Mức 4 — Thảm Họa: State File Mất Hoàn Toàn

File bị xóa và không có backup.

---

## Quy Trình Phục Hồi Từng Mức Độ

### Mức 1: Xử Lý Resource Drift

```bash
# Tùy chọn A: Xóa resource khỏi state (nếu tài nguyên thực sự đã mất)
terraform state rm aws_instance.web
# Sau đó terraform plan sẽ thấy resource cần tạo lại

# Tùy chọn B: Tạo lại tài nguyên và import lại
aws ec2 run-instances ...           # Tạo lại instance
terraform import aws_instance.web i-newid123   # Import vào state

# Tùy chọn C: Nếu resource vẫn tồn tại nhưng đã đổi ID
terraform state rm aws_instance.web
terraform import aws_instance.web i-correctid
```

### Mức 2: Phục Hồi Partial Apply

```bash
# Bước 1: KHÔNG chạy apply thêm ngay — kiểm tra tình trạng trước
terraform state list
terraform plan   # Xem Terraform nghĩ gì cần làm tiếp

# Bước 2: Với mỗi resource bị dở dang:
# - Nếu resource đã được tạo trên cloud nhưng chưa trong state:
terraform import aws_resource.name <actual-id>

# - Nếu resource trong state nhưng chưa tạo được trên cloud:
terraform state rm aws_resource.name
# Sau đó apply lại

# Bước 3: Sau khi state nhất quán, apply lại
terraform plan   # Verify plan trông đúng
terraform apply
```

### Mức 3: Phục Hồi JSON Corrupt

```bash
# Bước 1: Thử lấy state từ backup tự động (S3 versioning)
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix path/to/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Bước 2: Khôi phục version gần nhất hoạt động tốt
aws s3api get-object \
  --bucket my-terraform-state \
  --key path/to/terraform.tfstate \
  --version-id <VERSION_ID> \
  restored-state.json

# Bước 3: Kiểm tra state vừa khôi phục
cat restored-state.json | jq .

# Bước 4: Push state đã phục hồi lên backend
terraform state push restored-state.json

# Bước 5: Verify
terraform state list
terraform plan   # Plan phải không có thay đổi lớn bất ngờ
```

### Mức 4: Phục Hồi Khi Mất Hoàn Toàn State

```bash
# Đây là tình huống nghiêm trọng nhất — cần rebuild state từ đầu

# Bước 1: Tạo state file mới rỗng
echo '{"version": 4, "terraform_version": "1.5.0", "serial": 1, "lineage": "'$(uuidgen)'", "outputs": {}, "resources": []}' > new-state.json

# Bước 2: Push state rỗng
terraform state push new-state.json

# Bước 3: Import từng resource hiện có trên cloud
# Lấy danh sách tất cả resource đang chạy từ AWS Console / CLI
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --output text

# Bước 4: Import từng resource
terraform import aws_instance.web i-abc123
terraform import aws_s3_bucket.data my-bucket-name
terraform import aws_db_instance.postgres my-db-identifier
# ... tiếp tục cho từng resource

# Bước 5: Chạy plan và verify không có unexpected changes
terraform plan
```

---

## Dùng State Backup Tự Động

### Cấu Hình S3 Versioning — Quan Trọng Nhất

```hcl
# Trong S3 bucket lưu state — BẮT BUỘC phải bật versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"   # Không bao giờ tắt cái này
  }
}
```

### Script Backup Tự Động Trước Apply

```bash
#!/bin/bash
# scripts/safe-apply.sh

BUCKET="my-terraform-state"
KEY="env/prod/terraform.tfstate"
BACKUP_PREFIX="backups/$(date +%Y%m%d_%H%M%S)"

echo "=== Backup state trước khi apply ==="

# 1. Pull state hiện tại
terraform state pull > /tmp/pre-apply-backup.json

# 2. Upload backup lên S3 với timestamp
aws s3 cp /tmp/pre-apply-backup.json \
  "s3://${BUCKET}/${BACKUP_PREFIX}-terraform.tfstate"

echo "Backup tại: s3://${BUCKET}/${BACKUP_PREFIX}-terraform.tfstate"

# 3. Chạy plan
echo "=== Terraform Plan ==="
terraform plan -out=tfplan.binary

# 4. Yêu cầu xác nhận
echo ""
read -p "Apply plan trên? (yes/no): " confirm
if [ "$confirm" = "yes" ]; then
  terraform apply tfplan.binary
else
  echo "Apply bị hủy."
fi
```

---

## Ngăn Ngừa State Corruption

### Checklist Bắt Buộc

```hcl
# 1. Luôn dùng remote backend với versioning
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"   # State locking
    encrypt        = true                      # Mã hóa at rest
  }
}
```

```bash
# 2. KHÔNG BAO GIỜ interrupt apply đang chạy
# Nếu phải dừng khẩn cấp: đợi apply xong rồi rollback
# Dùng Ctrl+C là phương án cuối cùng

# 3. Chỉ có một người apply cùng lúc
# Dùng Atlantis hoặc Terraform Cloud để đảm bảo serial execution
# (thực thi tuần tự — một lần một)

# 4. Review plan kỹ trước apply
terraform plan -out=tfplan.binary    # Save plan
terraform show tfplan.binary         # Review chi tiết
terraform apply tfplan.binary        # Apply đúng plan đã review

# 5. Không chỉnh sửa state file bằng tay
# Dùng các lệnh: terraform state mv, rm, import
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Nếu state file bị xóa, bạn làm gì?**

A: Đây là sự cố mức 4 — nghiêm trọng nhất. Tôi sẽ:
1. Kiểm tra S3 versioning để tìm backup gần nhất
2. Nếu có backup: khôi phục và push lại
3. Nếu không có backup: tạo state rỗng và import từng resource từ cloud
4. Sau khi phục hồi: bật S3 versioning ngay lập tức và tạo runbook để tránh tái diễn

**Q: Làm sao phân biệt drift với corrupt?**

A: 
- **Drift** — Lệch: State hợp lệ nhưng không khớp với thực tế trên cloud. `terraform plan` sẽ cho thấy unexpected changes.
- **Corrupt** — Hỏng: State file không đọc được hoặc có dữ liệu sai format. Terraform báo lỗi khi đọc state.

---

## Tóm Tắt

```
State Corruption Severity:
├── Mức 1 (Drift)           → terraform state rm + import
├── Mức 2 (Partial Apply)   → Reconcile state thủ công rồi apply lại
├── Mức 3 (JSON Corrupt)    → Khôi phục từ S3 versioning backup
└── Mức 4 (Mất hoàn toàn)  → Rebuild từ đầu với terraform import

Ngăn ngừa:
├── Remote backend với S3 versioning BẬT
├── State locking với DynamoDB
├── Không interrupt apply
└── Backup trước mỗi apply quan trọng
```

---

**Xem Thêm:**
- [`02-state-management/3-state-locking.md`](../02-state-management/3-state-locking.md) — State Locking chi tiết
- [`02-state-management/5-state-recovery.md`](../02-state-management/5-state-recovery.md) — Recovery nâng cao
- [`5-debug-mode.md`](./5-debug-mode.md) — Debug khi không rõ nguyên nhân
