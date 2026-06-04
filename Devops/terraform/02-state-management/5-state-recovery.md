# 5 — State Recovery — Phục Hồi Khi State Bị Corrupt

> State corrupt — State hỏng — là một trong những sự cố nghiêm trọng nhất trong vận hành Terraform. Bài này hướng dẫn cách phòng ngừa, phát hiện sớm, và phục hồi khi xảy ra sự cố.

---

## 📚 Mục Lục

1. [Nguyên Nhân State Bị Corrupt](#1-nguyên-nhân-state-bị-corrupt)
2. [Nhận Biết State Có Vấn Đề](#2-nhận-biết-state-có-vấn-đề)
3. [Chiến Lược Phòng Ngừa](#3-chiến-lược-phòng-ngừa)
4. [Phục Hồi Từ S3 Versioning](#4-phục-hồi-từ-s3-versioning)
5. [Phục Hồi Bằng Cách Rebuild State](#5-phục-hồi-bằng-cách-rebuild-state)
6. [Xử Lý Trường Hợp Mất State Hoàn Toàn](#6-xử-lý-trường-hợp-mất-state-hoàn-toàn)
7. [Runbook — Quy Trình Xử Lý Sự Cố State](#7-runbook--quy-trình-xử-lý-sự-cố-state)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Nguyên Nhân State Bị Corrupt

### Nhóm 1: Lỗi Process — Sự Cố Quá Trình

```
✗ CI/CD runner bị kill giữa chừng khi apply (OOM, timeout)
✗ Network connection mất khi đang ghi state lên remote backend
✗ Laptop tắt đột ngột khi dùng local state
✗ Terraform crash do bug trong provider
```

### Nhóm 2: Lỗi Con Người — Human Error

```
✗ Force unlock khi apply đang chạy thực sự
✗ Dùng terraform state push với state cũ (ghi đè state mới)
✗ Hai người cùng apply vào local state (không có locking)
✗ Xóa nhầm state file trên S3
✗ Merge conflict trong state file được commit vào Git
```

### Nhóm 3: Lỗi Hạ Tầng — Infrastructure Issues

```
✗ S3 bucket bị xóa hoặc bị restricted policy đột ngột
✗ DynamoDB table bị xóa → không có locking → race condition
✗ Region outage — Sự cố vùng — khi apply đang chạy
```

---

## 2. Nhận Biết State Có Vấn Đề

### Dấu Hiệu 1: Terraform Muốn Destroy Mà Không Có Lý Do

```
Bạn không thay đổi gì trong code nhưng:
  Plan output: 15 resources to destroy, 15 resources to add
  → Terraform thấy resources trong state không còn khớp với thực tế
  → Hoặc state bị rollback về trạng thái cũ
```

### Dấu Hiệu 2: Parse Error Khi Đọc State

```bash
$ terraform plan
╷
│ Error: Failed to read state file
│
│ Error reading state: state file was created by a newer version of Terraform
│ (1.7.5); please upgrade to use it
╵

# Hoặc:
╷
│ Error: Invalid state file
│
│ The state file cannot be parsed as JSON.
╵
```

### Dấu Hiệu 3: Resource Tồn Tại Trên Cloud Nhưng Không Có Trong State

```bash
$ terraform state list | grep aws_rds
# Không có output → state không biết về RDS

$ aws rds describe-db-instances --query 'DBInstances[*].DBInstanceIdentifier'
# Nhưng RDS vẫn tồn tại trên AWS → state bị mất hoặc corrupt
```

### Dấu Hiệu 4: Serial Number Bất Thường

```bash
# Kiểm tra serial của state
terraform state pull | jq '.serial'
# Nếu serial đột ngột giảm → state đang dùng là bản cũ hơn
```

---

## 3. Chiến Lược Phòng Ngừa

### Bắt Buộc (Non-negotiable)

```hcl
# 1. Remote backend với versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"   # GIỮ LỊCH SỬ STATE
  }
}

# 2. State Locking luôn bật
terraform {
  backend "s3" {
    dynamodb_table = "terraform-state-locks"  # KHÔNG BAO GIỜ BỎ QUA
  }
}

# 3. Encryption at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### Quy Trình Trước Apply Lớn

```bash
# Backup state thủ công trước các thao tác nguy hiểm
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
terraform state pull > "state_backup_${TIMESTAMP}.tfstate"

# Lưu vào S3 với tag rõ ràng
aws s3 cp "state_backup_${TIMESTAMP}.tfstate" \
  "s3://mycompany-backups/terraform-state-backups/prod_${TIMESTAMP}.tfstate"
```

### Lifecycle Rule — Giữ Backup Đủ Lâu

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "state_backup" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "retain-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90   # Giữ 90 ngày lịch sử — đủ để rollback
    }
  }
}
```

---

## 4. Phục Hồi Từ S3 Versioning

Đây là kịch bản phổ biến nhất và dễ xử lý nhất khi đã bật versioning.

### Bước 1: Xác Định Version Cần Rollback

```bash
# Liệt kê tất cả versions của state file
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix prod/vpc/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified,IsLatest:IsLatest}' \
  --output table

# Output:
# VersionId              | LastModified              | IsLatest
# abc123xyz...           | 2026-05-12T10:00:00.000Z  | True     ← Bị corrupt
# def456uvw...           | 2026-05-11T15:30:00.000Z  | False    ← Version tốt
# ghi789rst...           | 2026-05-10T09:00:00.000Z  | False
```

### Bước 2: Tải Version Tốt Về Kiểm Tra

```bash
# Tải version cũ về
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key prod/vpc/terraform.tfstate \
  --version-id def456uvw \
  recovered_state.tfstate

# Kiểm tra nội dung
cat recovered_state.tfstate | jq '.serial, (.resources | length)'
# Serial phải thấp hơn version bị corrupt
# Số resources phải gần đúng với mong đợi
```

### Bước 3: Restore Version

**Cách 1: Xóa version mới nhất (bị corrupt) để version trước trở thành latest**

```bash
# Xóa version bị corrupt
aws s3api delete-object \
  --bucket mycompany-terraform-state \
  --key prod/vpc/terraform.tfstate \
  --version-id abc123xyz

# Version tốt nhất bây giờ tự động trở thành latest
```

**Cách 2: Push state tốt lên thay thế**

```bash
# Tải version tốt về
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key prod/vpc/terraform.tfstate \
  --version-id def456uvw \
  recovered_state.tfstate

# Đẩy state tốt lên (Terraform sẽ tăng serial tự động)
terraform state push recovered_state.tfstate
```

### Bước 4: Verify State Sau Khi Restore

```bash
# Chạy plan — xem Terraform muốn thay đổi gì
terraform plan

# Nếu plan thấy nhiều thứ cần thay đổi → state vẫn lệch
# Nếu plan gần như "No changes" → recovery thành công
```

---

## 5. Phục Hồi Bằng Cách Rebuild State

Khi không có backup, cần rebuild state từ đầu bằng cách import từng resource.

### Bước 1: Liệt Kê Resources Thực Tế Trên Cloud

```bash
# AWS: Liệt kê tất cả EC2 instances đang chạy
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=terraform" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table

# AWS: Liệt kê S3 buckets
aws s3 ls

# GCP: Liệt kê instances
gcloud compute instances list --format="table(name,zone,status)"
```

### Bước 2: Tạo Empty State

```bash
# Nếu state file bị corrupt hoàn toàn, xóa để bắt đầu lại
# CẢNH BÁO: Chỉ làm khi chắc chắn state không thể recover
terraform state push /dev/null   # Không dùng cách này

# Cách đúng: Tạo state rỗng
echo '{"version": 4, "terraform_version": "1.7.0", "serial": 1, "lineage": "'$(uuidgen)'", "outputs": {}, "resources": []}' > empty.tfstate
terraform state push empty.tfstate
```

### Bước 3: Import Từng Resource

```bash
# Với mỗi resource trong code, chạy import
terraform import aws_vpc.main vpc-0abc123def
terraform import aws_subnet.private["us-east-1a"] subnet-0abc123def
terraform import aws_subnet.private["us-east-1b"] subnet-0def456ghi
terraform import aws_instance.web i-0abc123def456789

# Hoặc dùng script để import hàng loạt
```

### Script Import Hàng Loạt — Ví Dụ

```bash
#!/bin/bash
# import_ec2.sh — Import tất cả EC2 instances theo tag

set -e

# Lấy danh sách instances có tag ManagedBy=terraform
INSTANCES=$(aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=terraform" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

for instance_id in $INSTANCES; do
  # Lấy tên từ tag Name
  name=$(aws ec2 describe-instances \
    --instance-ids "$instance_id" \
    --query 'Reservations[0].Instances[0].Tags[?Key==`TerraformName`].Value' \
    --output text)
  
  echo "Importing aws_instance.$name ($instance_id)..."
  terraform import "aws_instance.$name" "$instance_id"
done

echo "Import complete. Running plan to verify..."
terraform plan
```

---

## 6. Xử Lý Trường Hợp Mất State Hoàn Toàn

Đây là kịch bản tệ nhất — state file bị xóa hoàn toàn và không có backup.

### Đánh Giá Thiệt Hại

```bash
# 1. Kiểm tra tài nguyên nào đang tồn tại trên cloud
# 2. So sánh với code Terraform
# 3. Quyết định chiến lược: Import toàn bộ hay rebuild từ đầu
```

### Chiến Lược 1: Import Toàn Bộ (Ưu Tiên)

```
Ưu điểm: Giữ nguyên tài nguyên, không downtime
Nhược điểm: Tốn thời gian, phải import đúng IDs
Khi nào: Production environment, tài nguyên stateful (databases, volumes)
```

### Chiến Lược 2: Destroy và Recreate (Chỉ Khi Được Phép)

```
Ưu điểm: Clean state, đơn giản
Nhược điểm: Downtime, mất data trong stateful resources
Khi nào: Dev/test environment, stateless resources, không có data quan trọng
```

### Chiến Lược 3: Parallel Environment

```
Tạo environment mới song song → Migrate traffic → Destroy old
Ưu điểm: Không downtime, có thể test trước
Nhược điểm: Tốn chi phí chạy double environment
Khi nào: Production, không thể downtime, có thể migrate data
```

---

## 7. Runbook — Quy Trình Xử Lý Sự Cố State

```
╔══════════════════════════════════════════════════════════════╗
║           STATE INCIDENT RUNBOOK — Quy Trình Xử Lý          ║
╚══════════════════════════════════════════════════════════════╝

BƯỚC 1: DỪNG MỌI APPLY NGAY LẬP TỨC
  □ Thông báo team: "Freeze Terraform — không ai apply"
  □ Cancel CI/CD pipelines đang chạy
  □ Kiểm tra có apply nào đang lock state không

BƯỚC 2: ĐÁNH GIÁ MỨC ĐỘ SỰ CỐ
  □ State bị corrupt một phần hay hoàn toàn?
  □ Có backup không? (S3 versioning, manual backup)
  □ Hạ tầng thực tế còn nguyên không?
  □ Có apply nào đang chạy dở không?

BƯỚC 3: BACKUP HIỆN TRẠNG
  □ terraform state pull > incident_$(date +%Y%m%d_%H%M%S).tfstate
  □ Chụp ảnh toàn bộ state hiện tại (dù bị corrupt)
  □ Ghi lại mọi thay đổi đã thực hiện

BƯỚC 4: CHỌN CHIẾN LƯỢC PHỤC HỒI
  □ Nếu có backup gần nhất: → Restore từ backup (Bước 5a)
  □ Nếu S3 versioning bật: → Rollback version (Bước 5b)
  □ Nếu không có backup: → Import rebuild (Bước 5c)

BƯỚC 5a: RESTORE TỪ BACKUP
  □ Tải backup xuống
  □ Verify nội dung (serial, resource count)
  □ terraform state push backup.tfstate
  □ terraform plan → xem diff

BƯỚC 5b: ROLLBACK S3 VERSION
  □ Liệt kê versions: aws s3api list-object-versions ...
  □ Xác định version tốt gần nhất
  □ Xóa version corrupt hoặc push version tốt
  □ terraform plan → verify

BƯỚC 5c: IMPORT REBUILD
  □ Liệt kê tất cả resources trên cloud
  □ Tạo empty state
  □ Import từng resource
  □ terraform plan → "No changes" là thành công

BƯỚC 6: VERIFY VÀ DOCUMENT
  □ terraform plan → "No changes" hoặc chỉ có expected changes
  □ Chạy một apply nhỏ để test
  □ Ghi lại root cause — nguyên nhân gốc rễ
  □ Cập nhật runbook nếu cần
  □ Thực hiện post-mortem — rút kinh nghiệm
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q: Làm gì khi state file bị corrupt trong production?

**Trả lời mẫu:**
> Bước đầu tiên là freeze — đóng băng — mọi Terraform operations và thông báo team. Sau đó đánh giá mức độ: state bị corrupt một phần hay hoàn toàn, hạ tầng thực tế còn nguyên không. Nếu dùng S3 backend với versioning — tôi luôn bật — thì rollback về version tốt gần nhất. Nếu không có backup, phải rebuild state bằng cách import từng resource từ cloud. Sau khi recover, chạy `terraform plan` để verify state khớp với thực tế. Cuối cùng, làm post-mortem để tìm root cause và ngăn tái diễn.

### Q: Làm sao để phòng tránh state corruption?

**Trả lời mẫu:**
> Ba lớp phòng vệ: (1) Remote backend với State Locking — S3+DynamoDB hoặc Terraform Cloud — ngăn race condition; (2) S3 versioning với lifecycle rule giữ 90 ngày lịch sử — phục hồi dễ dàng; (3) Quy trình team: không ai apply bằng tay trong production, mọi thứ qua CI/CD với timeout và retry logic đúng đắn. Tôi cũng recommend backup state thủ công trước các thao tác lớn như module refactor hay infrastructure migration.

### Q: Terraform state push nguy hiểm như thế nào và khi nào được dùng?

**Trả lời mẫu:**
> `terraform state push` rất nguy hiểm vì nó ghi đè state hiện tại mà không có confirmation hay rollback tự động. Nó phù hợp trong hai trường hợp: (1) Phục hồi state từ backup sau sự cố — đây là mục đích thiết kế của lệnh này; (2) Migrate state giữa backends thủ công trong trường hợp `terraform init -migrate-state` không khả dụng. Quy tắc an toàn: luôn backup state hiện tại trước khi push, verify serial number, và chạy `terraform plan` ngay sau để xác nhận state hợp lệ.

---

## 🔗 Liên Kết Liên Quan

- [3-state-locking.md](./3-state-locking.md) — Force unlock khi state bị khoá do crash
- [4-state-commands.md](./4-state-commands.md) — `state pull`, `state push`, `import`
- [../09-troubleshooting/1-state-corruption.md](../09-troubleshooting/1-state-corruption.md) — Xử lý sự cố state nâng cao

---

**Cập Nhật Lần Cuối:** 2026-05-12
