# Production Checklist — Kiểm Tra Trước Khi Apply Vào Production

> Checklist — Danh sách kiểm tra — này là hàng rào cuối cùng trước khi thay đổi hạ tầng production. Áp dụng MỖI LẦN trước khi apply, không có ngoại lệ.

---

## Tại Sao Cần Checklist?

Sự cố hạ tầng thường xảy ra không phải vì thiếu kiến thức, mà vì:
- Apply vội vàng không review kỹ plan output
- Quên kiểm tra blast radius — phạm vi ảnh hưởng
- Không có rollback plan — kế hoạch khôi phục
- Apply vào giờ cao điểm
- Không thông báo team trước khi thay đổi lớn

---

## Checklist Giai Đoạn 1: Trước Khi Tạo Plan

### 1.1 Kiểm Tra Môi Trường

```bash
# Xác nhận đang làm việc với đúng environment
terraform workspace show
echo "AWS Account: $(aws sts get-caller-identity --query Account --output text)"
echo "AWS Region: $AWS_DEFAULT_REGION"

# Với GCP
gcloud config get-value project
gcloud config get-value compute/region

# Với state backend
terraform state pull | jq '{workspace: .serial, environment: .lineage}' 2>/dev/null || echo "Kiểm tra backend config"
```

**Checklist:**
- [ ] Đang ở đúng workspace (prod, không phải dev/staging)
- [ ] Đang dùng đúng AWS account / GCP project / Azure subscription
- [ ] Backend S3/GCS bucket là của đúng environment
- [ ] Không có state lock đang tồn tại từ session trước

### 1.2 Kiểm Tra Code Changes

```bash
# Review những thay đổi trong code
git diff main...HEAD --name-only
git log main...HEAD --oneline

# Kiểm tra không có file nhạy cảm bị commit
git diff main...HEAD -- "*.tfvars" "*.env" "*secret*" "*credential*"
```

**Checklist:**
- [ ] Đã review toàn bộ code changes trong PR/MR
- [ ] Không có hardcoded credentials trong code
- [ ] Không có `.tfvars` file chứa sensitive data được commit
- [ ] Code đã qua review của ít nhất 1 người khác (4-eyes principle — nguyên tắc 4 mắt)
- [ ] CI/CD checks — kiểm tra tự động — đã pass (tfsec, Checkov, TFLint)

---

## Checklist Giai Đoạn 2: Review Plan

### 2.1 Tạo và Lưu Plan

```bash
# Tạo plan và lưu ra file
terraform plan -out=tfplan.binary -detailed-exitcode
PLAN_EXIT_CODE=$?

# Exit codes:
# 0 = No changes
# 1 = Error
# 2 = Changes present

if [ $PLAN_EXIT_CODE -eq 0 ]; then
  echo "✅ Không có thay đổi"
elif [ $PLAN_EXIT_CODE -eq 1 ]; then
  echo "❌ Có lỗi trong plan"
  exit 1
elif [ $PLAN_EXIT_CODE -eq 2 ]; then
  echo "⚠️ Có thay đổi — Review kỹ!"
fi

# Xem plan chi tiết
terraform show tfplan.binary

# Export sang JSON để phân tích
terraform show -json tfplan.binary > tfplan.json
```

### 2.2 Phân Tích Số Lượng Thay Đổi

```bash
# Đếm resource theo loại action
terraform show -json tfplan.binary | jq '
  [.resource_changes[].change.actions[]] |
  group_by(.) |
  map({action: .[0], count: length}) |
  .[]
'

# Kết quả mẫu:
# {"action": "create", "count": 5}
# {"action": "update", "count": 2}
# {"action": "delete", "count": 0}   ← Cảnh báo nếu có delete!
# {"action": "replace", "count": 1}  ← CỰC KỲ CẨN THẬN với replace!
```

**Checklist:**
- [ ] Xem toàn bộ plan output, không skip
- [ ] Đếm số lượng: **create**, **update**, **delete**, **replace**
- [ ] Mọi `delete` đều có lý do rõ ràng và được accept
- [ ] Mọi `replace` (delete + create) được review kỹ — downtime có thể xảy ra
- [ ] Không có thay đổi bất ngờ (unexpected changes) ngoài dự kiến

### 2.3 Kiểm Tra Các Thay Đổi Nguy Hiểm

```bash
# Tìm các thay đổi force-replace — buộc tạo lại
terraform show -json tfplan.binary | jq '
  .resource_changes[] |
  select(.change.actions == ["delete", "create"] or
         .change.actions == ["create", "delete"]) |
  {address: .address, reason: .action_reason}
'

# Tìm thay đổi trên database resources — CỰC KỲ NGUY HIỂM
terraform show -json tfplan.binary | jq '
  .resource_changes[] |
  select(.address | test("rds|database|db_instance|cluster"))
'

# Tìm thay đổi security groups — Có thể block traffic
terraform show -json tfplan.binary | jq '
  .resource_changes[] |
  select(.address | test("security_group|firewall|network_acl"))
'
```

**Checklist — CẦN XEM KỸ:**
- [ ] Không có database instance nào bị **replace** (gây mất dữ liệu)
- [ ] Không có VPC hoặc subnet bị **delete** hoặc **replace**
- [ ] Không có security group rules bị xóa sẽ block traffic production
- [ ] Không có IAM role/policy bị thay đổi sẽ break service
- [ ] Không có DNS record bị xóa (gây outage — gián đoạn dịch vụ)
- [ ] Mọi `lifecycle { prevent_destroy = true }` được set cho critical resources

---

## Checklist Giai Đoạn 3: Chuẩn Bị Trước Apply

### 3.1 Backup Quan Trọng

```bash
# 1. Backup Terraform state
terraform state pull > "backup/state-before-apply-$(date +%Y%m%d_%H%M%S).json"

# 2. Backup database (nếu có thay đổi database)
# AWS RDS
aws rds create-db-snapshot \
  --db-instance-identifier production-db \
  --db-snapshot-identifier "pre-terraform-$(date +%Y%m%d)"

# 3. Xác nhận S3 versioning đang bật
aws s3api get-bucket-versioning \
  --bucket my-terraform-state-bucket \
  --query 'Status'
# Phải trả về "Enabled"
```

**Checklist:**
- [ ] State file đã được backup lên S3 hoặc local
- [ ] Database backup đã được tạo nếu có thay đổi database
- [ ] Biết cách restore về trạng thái trước nếu cần

### 3.2 Xác Nhận Maintenance Window

**Checklist:**
- [ ] Giờ apply không phải giờ cao điểm (peak hours — giờ lưu lượng cao)
- [ ] Không có sự kiện kinh doanh quan trọng trong vòng 2 giờ tới
- [ ] Đã thông báo cho team và on-call engineer
- [ ] Change ticket đã được tạo và approved (nếu có process)
- [ ] Rollback plan — kế hoạch khôi phục — đã được chuẩn bị

### 3.3 Rollback Plan

```bash
# Ghi lại rollback plan TRƯỚC KHI apply:
cat << 'EOF' > /tmp/rollback-plan.md
## Rollback Plan cho Apply $(date)

### Nếu apply thất bại:
1. Không panic — kiểm tra error message
2. Chạy terraform plan lại để xem tình trạng hiện tại
3. Nếu state inconsistent: terraform state pull > corrupted.json
4. Khôi phục state từ backup: terraform state push backup/state-before-apply-XXXXXX.json
5. Liên hệ: [on-call engineer name] tại [phone/slack]

### Nếu apply thành công nhưng service bị ảnh hưởng:
1. Rollback code: git revert <commit>
2. Apply lại với code cũ
3. Hoặc dùng terraform state mv để đổi lại config
EOF
```

---

## Checklist Giai Đoạn 4: Trong Và Sau Apply

### 4.1 Monitor Trong Khi Apply

```bash
# Terminal 1: Chạy apply
terraform apply tfplan.binary

# Terminal 2: Monitor AWS CloudWatch
watch -n 5 "aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_ELB_5XX_Count \
  --start-time $(date -u -v-5M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --query 'Datapoints[*].Sum'"

# Terminal 3: Monitor application health
watch -n 10 "curl -s https://api.example.com/health | jq .status"
```

**Checklist trong khi apply:**
- [ ] Theo dõi error rate trong monitoring dashboard
- [ ] Kiểm tra application health endpoint
- [ ] Sẵn sàng interrupt — ngắt — nếu có dấu hiệu outage

### 4.2 Verify Sau Apply

```bash
# 1. Verify state nhất quán
terraform plan -detailed-exitcode
# Phải trả về exit code 0 (no changes)

# 2. Verify outputs
terraform output

# 3. Smoke test — Kiểm tra nhanh chức năng cơ bản
curl -s https://api.example.com/health
aws ec2 describe-instances --instance-ids <new-id> --query 'Reservations[*].Instances[*].State.Name'

# 4. Kiểm tra không có resource orphan — tài nguyên mồ côi
terraform state list | wc -l  # So sánh với trước apply
```

**Checklist sau apply:**
- [ ] `terraform plan` sau apply trả về "No changes"
- [ ] Application health check trả về healthy
- [ ] Error rate trong monitoring bình thường
- [ ] Rollback plan không cần dùng đến

---

## Template Thông Báo Trước Apply

```markdown
## 📢 Terraform Apply Notification — Thông Báo Apply Hạ Tầng

**Environment:** production
**Time:** 2026-05-12 14:00 UTC
**Author:** [Tên bạn]
**Ticket:** INFRA-1234

### Changes Summary — Tóm Tắt Thay Đổi
- create: 3 resources (aws_security_group_rule × 3)
- update: 1 resource (aws_launch_template.api)
- delete: 0 resources

### Risk Assessment — Đánh Giá Rủi Ro
- **Risk Level:** Low — Thấp
- **Downtime Expected:** None — Không có
- **Affected Services:** API service (rolling restart — khởi động lại cuốn chiếu)

### Rollback Plan — Kế Hoạch Khôi Phục
1. Revert commit XXXXXXX
2. Apply lại với code cũ
3. On-call: @oncall-engineer

@channel Nếu có vấn đề sau 14:00 UTC, hãy mention @oncall-engineer
```

---

## Checklist Tóm Tắt Một Trang

```
PRE-PLAN
□ Đúng workspace/account/region?
□ Code đã được review (4 eyes)?
□ CI/CD checks đã pass?

REVIEW PLAN
□ Đọc toàn bộ plan output
□ Số lượng changes hợp lý?
□ Không có unexpected deletes?
□ Không có database replace?
□ Không có security group rules bị xóa quan trọng?

PREPARE
□ State file đã backup?
□ Database snapshot đã tạo (nếu cần)?
□ Giờ apply OK, không phải peak hour?
□ Team đã được thông báo?
□ Rollback plan đã viết ra?

APPLY & VERIFY
□ Monitor error rate trong khi apply
□ terraform plan sau apply = "No changes"
□ Smoke test application OK?
□ Update ticket/runbook sau khi xong?
```

---

## Câu Hỏi Phỏng Vấn

**Q: Quy trình của bạn trước khi apply Terraform vào production là gì?**

A: Tôi theo quy trình 4 giai đoạn:
1. **Pre-plan**: Verify đúng workspace/account, code đã review và CI pass
2. **Plan review**: Đọc kỹ plan output, đặc biệt chú ý `replace` và `delete`, dùng `terraform show -json` để phân tích kỹ hơn
3. **Prepare**: Backup state file, tạo DB snapshot nếu cần, chọn maintenance window, viết rollback plan
4. **Apply & verify**: Monitor trong khi chạy, verify `terraform plan` sau apply = "No changes", smoke test application

Điều tôi coi là quan trọng nhất: không bao giờ skip bước review plan, đặc biệt với các thay đổi có `replace` action vì nó có thể gây downtime.

---

**Xem Thêm:**
- [`1-state-corruption.md`](./1-state-corruption.md) — Xử lý khi state bị hỏng
- [`06-cicd/5-rollback-strategy.md`](../06-cicd/5-rollback-strategy.md) — Chiến lược rollback chi tiết
- [`08-monitoring/1-drift-detection.md`](../08-monitoring/1-drift-detection.md) — Phát hiện drift sau apply
