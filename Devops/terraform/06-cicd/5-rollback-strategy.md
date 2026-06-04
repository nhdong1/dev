# Rollback Strategy — Chiến Lược Khôi Phục Khi Có Sự Cố

> Rollback — Khôi Phục Về Trạng Thái Trước — là khả năng đưa hạ tầng về trạng thái hoạt động tốt trước đó khi một lần apply gây ra sự cố. Trong Terraform, rollback không đơn giản như ứng dụng vì hạ tầng có trạng thái — stateful.

---

## 🎯 Mục Tiêu

- Hiểu tại sao rollback hạ tầng phức tạp hơn rollback ứng dụng
- Thiết kế strategy rollback cho từng loại sự cố
- Áp dụng `moved` block và `terraform import` khi cần
- Xây dựng runbook — Tài liệu quy trình xử lý sự cố — cho team
- Phòng ngừa sự cố bằng các biện pháp trước khi apply

---

## 🔴 Tại Sao Rollback Hạ Tầng Khó?

```
Ứng dụng (đơn giản):
git revert → build → deploy → done

Hạ tầng (phức tạp):
Vấn đề 1: Một số tài nguyên không thể "undo" (S3 bucket với data, RDS)
Vấn đề 2: Destroy và recreate có thể gây downtime
Vấn đề 3: Terraform state có thể không đồng bộ với thực tế
Vấn đề 4: Dependencies phức tạp giữa các tài nguyên
Vấn đề 5: Một số thay đổi là destructive — Phá hủy — không phục hồi được
```

---

## 📊 Phân Loại Sự Cố và Strategy Tương Ứng

### Mức 1: Lỗi Cấu Hình Nhỏ

```
Ví dụ: Đổi sai port, sai tên tag, sai environment variable

Triệu chứng: Service không hoạt động đúng nhưng hạ tầng vẫn còn

Rollback:
1. Revert commit trong git
2. terraform plan (xem có gì thay đổi)
3. terraform apply (apply thay đổi ngược lại)

Thời gian: 5-15 phút
Rủi ro: Thấp
```

### Mức 2: Tài Nguyên Bị Thay Thế — Resource Replacement

```
Ví dụ: Thay đổi AMI của EC2, thay đổi cấu hình subnet của RDS

Triệu chứng: Terraform plan cho thấy "must be replaced"
             Tài nguyên bị destroy và recreate

Rollback:
1. Dùng terraform state để kiểm tra
2. Revert commit git
3. terraform plan — xác nhận plan sẽ recreate lại
4. terraform apply — accept downtime hoặc dùng blue/green

Thời gian: 15-60 phút (tùy loại tài nguyên)
Rủi ro: Trung bình (có downtime)
```

### Mức 3: Mất Dữ Liệu — Data Loss

```
Ví dụ: Xóa nhầm RDS, xóa S3 bucket, drop table

Triệu chứng: Dữ liệu không còn sau khi apply

Rollback:
1. KHÔNG thể rollback bằng Terraform
2. Restore từ backup (RDS snapshot, S3 versioning)
3. Cập nhật Terraform code để phản ánh đúng thực tế

Thời gian: Vài giờ đến vài ngày
Rủi ro: Cao (mất dữ liệu có thể vĩnh viễn)
```

### Mức 4: State Corruption — State Bị Hỏng

```
Ví dụ: State file không đồng bộ với thực tế sau khi apply một phần

Triệu chứng: terraform plan cho kết quả kỳ lạ
             Tài nguyên tồn tại trên cloud nhưng không có trong state

Rollback:
1. terraform state list (xem state hiện tại)
2. terraform import (nhập lại tài nguyên vào state)
3. Hoặc restore state từ backup

Xem chi tiết: 02-state-management/5-state-recovery.md
```

---

## 🛡️ Phòng Ngừa Sự Cố — Prevention First

### Lifecycle Rules — Quy Tắc Vòng Đời

```hcl
# Ngăn xóa nhầm tài nguyên quan trọng
resource "aws_db_instance" "production" {
  identifier = "prod-postgres"
  # ...

  lifecycle {
    # Terraform sẽ báo lỗi nếu muốn xóa resource này
    prevent_destroy = true

    # Tạo resource mới trước khi xóa cái cũ
    # Dùng khi không chấp nhận downtime
    create_before_destroy = true

    # Bỏ qua thay đổi của những fields này
    # (ví dụ: password được quản lý bên ngoài Terraform)
    ignore_changes = [password, snapshot_identifier]
  }
}
```

### State Backup — Sao Lưu State

```hcl
# S3 backend với versioning bật sẵn
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-southeast-1"

    # Versioning cho phép restore state cũ
    # Bật versioning ở S3 bucket level, không phải ở đây
  }
}
```

```bash
# Bật versioning cho S3 bucket state
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

# Bật MFA Delete để ngăn xóa version lịch sử
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/my-mfa 123456"
```

### Manual Checkpoint — Lưu State Thủ Công Trước Khi Apply Lớn

```bash
#!/bin/bash
# backup-state.sh — Chạy trước khi apply thay đổi lớn

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ENVIRONMENT=${1:-production}

echo "📋 Tạo backup state trước khi apply..."

# Lưu state hiện tại ra file local
terraform state pull > "backups/state_${ENVIRONMENT}_${TIMESTAMP}.json"

# Tạo plan file để review
terraform plan \
  -var-file="environments/${ENVIRONMENT}.tfvars" \
  -out="backups/plan_${ENVIRONMENT}_${TIMESTAMP}.tfplan"

echo "✅ Backup tạo tại: backups/state_${ENVIRONMENT}_${TIMESTAMP}.json"
echo "📄 Plan saved tại: backups/plan_${ENVIRONMENT}_${TIMESTAMP}.tfplan"
echo ""
echo "Để rollback: terraform state push backups/state_${ENVIRONMENT}_${TIMESTAMP}.json"
```

---

## 🔄 Kỹ Thuật Rollback Cụ Thể

### Kỹ Thuật 1: Git Revert + Re-apply

```bash
# Bước 1: Xác định commit gây sự cố
git log --oneline -10

# Bước 2: Revert commit đó (tạo commit mới, không xóa history)
git revert <commit-hash>

# Bước 3: Xem plan để confirm
terraform plan

# Bước 4: Apply nếu plan đúng
terraform apply
```

```
Khi nào dùng:
✅ Thay đổi gần đây gây ra sự cố
✅ Không có dữ liệu bị mất
✅ Tài nguyên có thể quay về cấu hình cũ mà không cần destroy/recreate
```

### Kỹ Thuật 2: Restore State Từ S3 Versioning

```bash
# Bước 1: List các version của state file
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix production/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified}' \
  --output table

# Bước 2: Download version cũ
aws s3api get-object \
  --bucket my-terraform-state \
  --key production/terraform.tfstate \
  --version-id "<version-id-cần-restore>" \
  old-state.json

# Bước 3: Review nội dung state cũ
cat old-state.json | jq '.resources | length'

# Bước 4: Push state cũ lên (NGUY HIỂM — đảm bảo team không đang làm gì)
terraform state push old-state.json

# Bước 5: Kiểm tra
terraform plan  # Xem Terraform muốn làm gì với state mới
```

```
⚠️ Cảnh báo: Push state cũ có thể khiến Terraform không biết
về các tài nguyên đã được tạo sau đó.
Cần terraform import hoặc terraform state rm để sync lại.
```

### Kỹ Thuật 3: Selective State Manipulation

```bash
# Tình huống: Một resource bị xóa nhầm khỏi state
# nhưng vẫn còn tồn tại trên AWS

# Bước 1: Kiểm tra tài nguyên còn trên AWS không
aws ec2 describe-instances --instance-ids i-1234567890abcdef0

# Bước 2: Import tài nguyên vào state
terraform import aws_instance.web_server i-1234567890abcdef0

# Bước 3: Verify
terraform plan  # Nên cho thấy "No changes" nếu config đúng
```

```bash
# Tình huống: Resource có trong state nhưng bị xóa trên cloud
# (ai đó xóa thủ công)

# Bước 1: Xem resource nào đang "drift"
terraform plan  # Sẽ thấy Terraform muốn recreate

# Bước 2a: Nếu muốn recreate
terraform apply  # Cho phép Terraform recreate

# Bước 2b: Nếu không muốn resource này nữa
terraform state rm aws_instance.old_server
# Sau đó xóa khỏi code
```

### Kỹ Thuật 4: Blue/Green Deployment Với Terraform

```hcl
# Chiến lược: Tạo mới trước, chuyển traffic, xóa cái cũ

# Bước 1: Tạo "green" environment
resource "aws_instance" "app_green" {
  ami           = var.new_ami
  instance_type = "t3.small"
  tags = {
    Name = "app-green"
    Color = "green"
  }
}

# Bước 2: Load balancer chuyển sang green
resource "aws_lb_target_group_attachment" "app_green" {
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.app_green.id
}

# Bước 3: Sau khi verify green hoạt động, xóa blue
# (Trong lần apply tiếp theo, xóa resource "app_blue")
```

---

## 📋 Runbook Xử Lý Sự Cố — Incident Runbook

### Phân Loại Mức Độ Nghiêm Trọng

```
P0 — Production hoàn toàn không hoạt động:
├── Thời gian phản hồi: Ngay lập tức
├── Người xử lý: On-call engineer + Lead
├── Kênh liên lạc: Phone + PagerDuty
└── Mục tiêu: Restore trong 30 phút

P1 — Production bị ảnh hưởng nghiêm trọng:
├── Thời gian phản hồi: 15 phút
├── Người xử lý: On-call engineer
├── Kênh liên lạc: Slack + PagerDuty
└── Mục tiêu: Restore trong 2 giờ

P2 — Một tính năng bị ảnh hưởng:
├── Thời gian phản hồi: 1 giờ
├── Người xử lý: Team engineer
├── Kênh liên lạc: Slack
└── Mục tiêu: Fix trong 8 giờ
```

### Quy Trình Xử Lý Chuẩn

```
1. PHÁT HIỆN (Detection) — 0-5 phút
   ├── Monitoring alert / user report
   ├── Xác định tầm ảnh hưởng
   └── Thông báo kênh incident

2. ĐÁNH GIÁ (Assessment) — 5-15 phút
   ├── Xem recent deployments (git log, CI history)
   ├── terraform state list
   ├── Kiểm tra cloud console
   └── Xác định nguyên nhân

3. KHÔI PHỤC (Recovery) — 15-60 phút
   ├── Chọn strategy phù hợp (xem bên trên)
   ├── Thực thi với ít nhất 2 người confirm
   └── Verify từng bước

4. XÁC NHẬN (Verification) — Sau khi fix
   ├── Kiểm tra health checks
   ├── Chạy smoke tests
   └── Thông báo stakeholders

5. POST-MORTEM — Trong 24-48 giờ
   ├── Timeline đầy đủ
   ├── Root cause analysis — Phân tích nguyên nhân gốc
   └── Action items để ngăn tái phát
```

---

## ⚡ Rollback Trong CI/CD Pipeline

### Automatic Rollback Sau Failed Apply

```yaml
# .github/workflows/terraform-apply.yml
- name: Terraform Apply
  id: apply
  run: terraform apply -auto-approve tfplan
  continue-on-error: true  # Không fail job ngay

# Nếu apply thất bại, thử rollback
- name: Rollback on Failure
  if: steps.apply.outcome == 'failure'
  run: |
    echo "⚠️ Apply thất bại. Thực hiện rollback..."

    # Restore state từ backup đã tạo trước apply
    if [ -f "pre-apply-state.json" ]; then
      terraform state push pre-apply-state.json
      echo "✅ State đã được restore về trạng thái trước apply"
    else
      echo "❌ Không tìm thấy backup state!"
      echo "Cần xử lý thủ công"
    fi

    # Thông báo team
    curl -X POST "$SLACK_WEBHOOK" \
      -d '{"text": "❌ TERRAFORM APPLY THẤT BẠI — Đang rollback. CC @oncall"}'

- name: Fail Pipeline After Rollback
  if: steps.apply.outcome == 'failure'
  run: exit 1
```

### State Backup Trước Mỗi Apply

```yaml
- name: Backup Current State
  run: |
    terraform state pull > pre-apply-state.json
    echo "State backup tạo: $(wc -c < pre-apply-state.json) bytes"

- name: Terraform Apply
  run: terraform apply -auto-approve tfplan
```

---

## 🔒 Ngăn Sự Cố Với Checklist Trước Apply

```bash
#!/bin/bash
# pre-apply-checklist.sh

echo "=== Checklist Trước Khi Apply Vào Production ==="
echo ""

# 1. Kiểm tra môi trường
echo "1. Bạn đang apply vào môi trường nào?"
terraform workspace show

# 2. Xem plan
echo ""
echo "2. Plan summary:"
terraform show tfplan | grep -E "^Plan:|will be|must be"

# 3. Kiểm tra các tài nguyên sẽ bị destroy
DESTROY_COUNT=$(terraform show tfplan | grep "will be destroyed" | wc -l)
if [ "$DESTROY_COUNT" -gt 0 ]; then
  echo ""
  echo "⚠️  CẢNH BÁO: ${DESTROY_COUNT} tài nguyên sẽ bị XÓA:"
  terraform show tfplan | grep "will be destroyed"
  echo ""
  read -p "Bạn có chắc chắn muốn tiếp tục? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    echo "Apply đã bị hủy."
    exit 1
  fi
fi

# 4. Backup state
echo ""
echo "3. Tạo backup state..."
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
terraform state pull > "backups/pre-apply-${TIMESTAMP}.json"
echo "   Backup: backups/pre-apply-${TIMESTAMP}.json"

# 5. Confirm cuối
echo ""
read -p "Xác nhận apply? (yes/no): " final_confirm
if [ "$final_confirm" != "yes" ]; then
  echo "Apply đã bị hủy."
  exit 1
fi

echo ""
echo "✅ Tiến hành apply..."
```

---

## 📊 Post-Mortem Template

```markdown
# Post-Mortem: [Tên Sự Cố] — [Ngày]

## Tóm Tắt

[1-2 câu mô tả sự cố]

## Timeline

| Thời Gian | Sự Kiện |
|-----------|---------|
| HH:MM     | Phát hiện sự cố |
| HH:MM     | Bắt đầu điều tra |
| HH:MM     | Xác định nguyên nhân |
| HH:MM     | Bắt đầu rollback |
| HH:MM     | Service hoạt động trở lại |

## Tầm Ảnh Hưởng

- Thời gian downtime: X phút
- Users bị ảnh hưởng: Y người
- Revenue impact: $Z (nếu có)

## Nguyên Nhân Gốc — Root Cause

[Mô tả chi tiết nguyên nhân]

## Quy Trình Xử Lý

1. [Bước 1]
2. [Bước 2]
3. [...]

## Điều Làm Tốt

- [Điểm tích cực 1]
- [Điểm tích cực 2]

## Điều Cần Cải Thiện

- [Vấn đề 1]
- [Vấn đề 2]

## Action Items

| Hành Động | Người Phụ Trách | Deadline |
|-----------|-----------------|----------|
| Thêm prevent_destroy vào RDS | @engineer | DD/MM |
| Tạo alarm cho state changes | @devops | DD/MM |
| Cập nhật runbook | @lead | DD/MM |
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Rollback Strategy

**Q: Terraform có tính năng "rollback" tự động không?**

A: Không. Terraform không có tính năng rollback tự động. Nếu apply thất bại giữa chừng, Terraform ở trạng thái partial — một số tài nguyên đã được tạo/sửa, một số chưa. Terraform sẽ cập nhật state file để phản ánh những gì đã thay đổi. Để "rollback", bạn phải:
1. Sửa code Terraform để phản ánh trạng thái muốn quay về
2. Chạy terraform apply lại
Hoặc restore state từ backup và để Terraform tự điều chỉnh.

**Q: Làm thế nào để chuẩn bị cho tình huống cần rollback khẩn cấp?**

A: Các biện pháp chuẩn bị:
1. Bật versioning cho S3 state bucket — luôn có thể restore state cũ
2. Backup state thủ công trước mỗi lần apply lớn
3. Dùng `prevent_destroy = true` cho tài nguyên quan trọng
4. Có runbook rõ ràng cho từng loại sự cố
5. Test rollback procedure trong staging trước
6. Maintain separate state per environment để sự cố production không ảnh hưởng staging

**Q: Bạn sẽ xử lý thế nào nếu Terraform apply bị interrupt giữa chừng?**

A: Đây là tình huống state corruption nguy hiểm. Bước xử lý:
1. Kiểm tra state lock còn không: `terraform force-unlock` nếu cần
2. Chạy `terraform state list` để xem state hiện tại
3. Vào cloud console để xem tài nguyên nào thực sự tồn tại
4. Dùng `terraform import` để nhập tài nguyên còn thiếu trong state
5. Dùng `terraform state rm` để xóa tài nguyên khỏi state nếu không còn tồn tại
6. Chạy `terraform plan` để verify state đã đồng bộ với thực tế
7. Apply nếu cần

---

## 📋 Checklist Rollback Strategy

### Chuẩn Bị Trước
- [ ] S3 state bucket có versioning bật
- [ ] Script backup state tự động chạy trước mỗi lần apply
- [ ] `prevent_destroy = true` cho databases và critical resources
- [ ] Runbook đã được viết và team đã đọc
- [ ] Đã test rollback trong staging ít nhất một lần

### Khi Sự Cố Xảy Ra
- [ ] Thông báo team ngay lập tức
- [ ] Xác định tầm ảnh hưởng trước khi hành động
- [ ] Backup state hiện tại TRƯỚC KHI làm bất kỳ thứ gì
- [ ] Có ít nhất 2 người confirm mỗi bước
- [ ] Document mọi hành động thực hiện

### Sau Sự Cố
- [ ] Verify service hoạt động bình thường
- [ ] Chạy smoke tests
- [ ] Viết post-mortem trong 48 giờ
- [ ] Implement action items để ngăn tái phát

---

**Hoàn Thành:** Bạn đã học xong toàn bộ `06-cicd/`

**Quay Lại:** [README.md](README.md) | **Phần Tiếp:** [07-testing/](../07-testing/)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
