# Câu Chuyện Sự Cố Theo Phương Pháp STAR

> STAR — Situation, Task, Action, Result — là framework kể chuyện phỏng vấn hành vi. Phần này cung cấp các câu chuyện mẫu thực tế về sự cố hạ tầng Terraform, kèm hướng dẫn cách tùy chỉnh cho kinh nghiệm của bạn.

---

## 📖 Hiểu Về Phương Pháp STAR

```
S — Situation  (Tình huống):  Bối cảnh, quy mô, constraints — ràng buộc
T — Task       (Nhiệm vụ):    Bạn cần đạt gì? Deadline? Trách nhiệm?
A — Action     (Hành động):   Bạn ĐÃ làm gì? Tại sao chọn cách đó?
R — Result     (Kết quả):     Kết quả định lượng được? Học được gì?
```

**Nguyên tắc vàng:**
- **A (Action)** là phần quan trọng nhất — chiếm 60% thời gian kể
- Kết quả cần có **số cụ thể** (thời gian downtime, % cải thiện, tiền tiết kiệm)
- Luôn có phần **"bài học rút ra"** — interviewer muốn thấy growth mindset

---

## 🔥 Câu Chuyện 1: State File Corruption trong Production

### Tình Huống (Situation)

> Startup e-commerce, team DevOps 3 người, hệ thống chạy trên AWS. Cuối năm, traffic tăng 5x, team đang vừa scale infrastructure vừa handle incidents liên tục.

**Setup cụ thể:**
- Production environment: 200+ AWS resources quản lý bởi Terraform
- Remote backend: S3 (nhưng chưa bật versioning — đây là sai lầm)
- CI/CD: GitHub Actions, 2 pipeline chạy song song

### Nhiệm Vụ (Task)

> Khi tôi đang apply một thay đổi tăng capacity cho Auto Scaling Group — Nhóm Tự Động Mở Rộng, đồng nghiệp đồng thời apply một thay đổi security group. Cả hai pipeline đều pass lock check (do race condition — điều kiện tranh chấp trong milliseconds), dẫn đến state file bị corrupt một phần.

**Hậu quả:**
- `terraform plan` báo lỗi JSON parse
- 15 resources bị orphaned — bị bỏ rơi — khỏi state
- Không thể apply bất kỳ thay đổi nào
- Đây là lúc 23:30, ngày trước Black Friday

### Hành Động (Action)

**Bước 1 — Stabilize — Ổn định (23:30–23:45):**
```bash
# Ngay lập tức freeze mọi pipeline
# Không ai được touch Terraform cho đến khi resolved

# Pull state hiện tại (dù hỏng) để backup
terraform state pull > corrupted-state-$(date +%Y%m%d-%H%M%S).json

# Xác nhận tài nguyên nào bị mất khỏi state
terraform state list 2>&1 | grep "Error" > missing-resources.txt
```

**Bước 2 — Assess Damage — Đánh Giá Thiệt Hại (23:45–00:15):**
```bash
# Lấy danh sách tài nguyên THỰC SỰ đang chạy trên AWS
aws ec2 describe-instances --filters "Name=tag:ManagedBy,Values=terraform" \
  --query 'Reservations[].Instances[].InstanceId' > actual-instances.txt

aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[].AutoScalingGroupName' > actual-asg.txt

# So sánh để xác định 15 orphaned resources
diff <(terraform state list 2>/dev/null) <(cat actual-resources.txt)
```

**Bước 3 — Recovery — Khôi Phục (00:15–02:30):**
```bash
# Không có S3 versioning → phải rebuild state từ thực tế
# Import từng resource một vào state mới

terraform import aws_autoscaling_group.main "production-asg-main"
terraform import aws_security_group.app sg-0abc123def456789a
# ... 13 resources còn lại

# Verify sau mỗi import
terraform plan -target=aws_autoscaling_group.main
# Phải thấy "No changes" nếu code khớp với thực tế
```

**Bước 4 — Hotfix Ngay Trong Đêm:**
```bash
# Bật S3 versioning ngay lập tức (đáng lẽ phải làm từ đầu)
aws s3api put-bucket-versioning \
  --bucket terraform-state-prod \
  --versioning-configuration Status=Enabled

# Thêm DynamoDB locking (trước đó không có)
# Cập nhật terraform backend config
```

### Kết Quả (Result)

- **Downtime:** Zero downtime cho end users — recovery diễn ra ở layer infrastructure, không ảnh hưởng running services
- **Recovery time:** 3 giờ để fully recover state (23:30 → 02:30)
- **Black Friday:** Hệ thống scale thành công, handle 5x traffic
- **Process improvement — Cải tiến quy trình:**
  - Bắt buộc S3 versioning trong terraform bootstrap module
  - Thêm concurrent pipeline protection trong CI/CD
  - Tạo runbook — Sổ tay xử lý sự cố — cho state recovery
  - Monthly state backup drill — Diễn tập khôi phục hàng tháng

**Bài học:**
> "Tôi học được rằng infrastructure best practices không phải optional. S3 versioning tốn $0.023/GB/tháng nhưng thiếu nó gây ra 3 giờ stressful work và risk production incident. Từ đó, mọi Terraform module tôi viết đều có S3 versioning là non-negotiable default."

---

## 🔥 Câu Chuyện 2: Accidental Deletion — Xóa Nhầm Database Production

### Tình Huống (Situation)

> Fintech startup, team 5 backend engineers kiêm infra. Database production chứa transaction data của 50,000 users. Tôi là người duy nhất trong team biết Terraform.

### Nhiệm Vụ (Task)

> Refactoring module structure: đổi tên `aws_db_instance.mysql` thành `aws_db_instance.primary` để consistent với naming convention mới. Tôi viết code rename nhưng quên thêm `moved` block hoặc `lifecycle prevent_destroy`.

**Điều xảy ra:**
```bash
terraform plan  # Output: "aws_db_instance.mysql will be destroyed"
                #         "aws_db_instance.primary will be created"
# Tôi KHÔNG đọc kỹ plan output — vội vàng apply
terraform apply  # ...đang xóa database production
```

Phát hiện sau 90 giây khi thấy application logs báo `cannot connect to database`.

### Hành Động (Action)

**T+0:00 — Phát Hiện và Dừng:**
```bash
# Ctrl+C apply ngay lập tức
# Nhưng RDS deletion thường không thể cancel sau khi bắt đầu
```

**T+0:02 — Kiểm Tra Snapshot:**
```bash
# AWS RDS có automated backup — sao lưu tự động
aws rds describe-db-snapshots \
  --query 'DBSnapshots[?Status==`available`]' \
  --output table

# Tìm snapshot gần nhất: 6 giờ trước
# RPO — Recovery Point Objective — thực tế: 6 giờ data
```

**T+0:05 — Restore từ Snapshot:**
```bash
# Restore sang instance mới (không thể restore vào cùng ID cũ)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier production-mysql-restored \
  --db-snapshot-identifier rds:production-mysql-2024-11-15-06-00

# RDS restore mất 10-20 phút
```

**T+0:05 — Parallel: Mitigate User Impact — Giảm Thiểu Ảnh Hưởng:**
```bash
# Update application config để show maintenance page
# Thông báo cho Customer Support team
# Xác định có transaction nào in-flight không
```

**T+0:25 — Database Available, Update Connection String:**
```bash
# Cập nhật Terraform code với endpoint mới
variable "db_endpoint" {
  default = "production-mysql-restored.xxx.rds.amazonaws.com"
}

# Import instance mới vào state
terraform import aws_db_instance.primary production-mysql-restored

# Apply để update security groups, parameter groups
terraform apply -target=aws_db_instance.primary
```

**T+0:30 — Application Back Online:**
- Total downtime: 28 phút

**T+1:00 — Identify Data Loss — Xác Định Mất Mát:**
```bash
# Backup: 06:00 AM
# Incident: 14:30 PM
# 8.5 giờ transaction data cần reconcile — Đối chiếu

# Từ application logs: 247 transactions trong khoảng trống
# Liên hệ 247 users để verify và replay manually
```

### Kết Quả (Result)

- **Downtime:** 28 phút (target SLA là 99.9% = 8.7 giờ/năm)
- **Data loss:** 247 transactions được recover manually trong 48 giờ tiếp theo
- **Financial impact:** Zero — không có transaction mất mát cuối cùng nhờ idempotent payment system
- **Process changes sau incident:**

```hcl
# 1. Mọi database đều có prevent_destroy
resource "aws_db_instance" "primary" {
  lifecycle {
    prevent_destroy = true
  }
}

# 2. Luôn dùng moved block khi rename
moved {
  from = aws_db_instance.mysql
  to   = aws_db_instance.primary
}

# 3. Checklist bắt buộc trước khi apply production
# - Plan output review bởi ít nhất 2 người
# - "destroy" keyword trong plan → mandatory second approval
```

**Bài học:**
> "Câu chuyện này dạy tôi rằng `terraform plan` phải được đọc như contract pháp lý — từng dòng. Hiện tại tôi có rule: bất kỳ plan nào có chữ 'destroy' đều cần peer review — review từ đồng nghiệp — không ngoại lệ. Và `moved` block đã trở thành reflex tự nhiên khi rename resource."

---

## 🔥 Câu Chuyện 3: Drift Recovery — Khôi Phục Sau Khi Lệch Cấu Hình Nghiêm Trọng

### Tình Huống (Situation)

> Scale-up startup, hệ thống AWS phức tạp với 500+ resources. Team DevOps vừa được nhận, hệ thống cũ được build hoàn toàn manual qua Console. Task của tôi: migrate — chuyển đổi — toàn bộ sang Terraform.

### Nhiệm Vụ (Task)

> 3 tháng import resources vào Terraform. Sau đó phát hiện: team security đã thay đổi 47 security groups trực tiếp qua Console trong 3 tháng đó mà không thông báo. Kết quả: `terraform plan` muốn REVERT 47 security groups về config trong code (cũ hơn và có thể insecure hơn).

### Hành Động (Action)

**Bước 1 — Assess Scale — Đánh Giá Quy Mô:**
```bash
# Chạy plan và export ra JSON để phân tích
terraform plan -json > plan.json

# Count số security group changes
cat plan.json | jq '[.resource_changes[] | 
  select(.type == "aws_security_group_rule" and .change.actions != ["no-op"])] 
  | length'
# Output: 234 changes (47 SGs × ~5 rules mỗi SG)
```

**Bước 2 — Triage — Phân Loại:**
```bash
# Phân loại thành 3 nhóm:
# Group A: Security rules added by security team → phải giữ
# Group B: Outdated rules Terraform muốn restore → bỏ đi
# Group C: Rules không ai nhớ tại sao có → cần review
```

**Bước 3 — Reconcile — Đối Chiếu:**
```hcl
# Với Group A: Cập nhật code để reflect actual state
# Với Group B: Giữ nguyên code, sẽ được applied (revert about-right)
# Với Group C: Schedule security review meeting

# Dùng terraform refresh cho Group A để sync state
terraform refresh -target=aws_security_group.app_server

# Hoặc manually update code từ terraform show output
terraform show -json | jq '.values.root_module.resources[] | 
  select(.address == "aws_security_group.app_server")'
```

**Bước 4 — Process Fix — Sửa Quy Trình:**
```
1. Họp với team security: mọi SG change phải qua Terraform PR
2. Setup AWS Config Rule: alert khi SG thay đổi ngoài Terraform
3. Bật CloudTrail → SNS → Slack: real-time notification khi manual change
4. Weekly drift scan: terraform plan -detailed-exitcode trong cron job
```

### Kết Quả (Result)

- **Reconciliation:** 3 tuần để clear tất cả drift
- **Prevention:** Zero manual console changes trong 6 tháng tiếp theo (nhờ alerting)
- **Cultural change:** Security team became Terraform users, không chỉ consumers
- **Tự động hóa:** Drift detection script chạy hàng ngày, alert trong 5 phút nếu có

---

## 📝 Template Tạo Câu Chuyện Của Bạn

Dùng template này để tạo STAR story từ kinh nghiệm thực tế:

```markdown
## Câu Chuyện: [Tên Ngắn Gọn]

### Tình Huống (Situation)
- Công ty/team có quy mô: ___
- Hệ thống: ___ resources trên ___ cloud provider
- Đặc điểm môi trường: production / staging / startup / enterprise
- Bối cảnh dẫn đến incident: ___

### Nhiệm Vụ (Task)
- Vấn đề cần giải quyết: ___
- Constraints — Ràng buộc: thời gian, resources, impact
- Trách nhiệm của tôi: ___

### Hành Động (Action)
1. Phát hiện vấn đề bằng cách nào? (tool, alert, manual discovery)
2. Triage — Đánh giá ưu tiên — initial assessment
3. Các bước giải quyết (có commands cụ thể nếu có thể)
4. Ai involved? Cách communicate với stakeholders?
5. Tại sao chọn approach này, không chọn approach khác?

### Kết Quả (Result)
- Thời gian resolution: ___
- Impact định lượng: ___ minutes downtime, ___ % improvement, $__ saved
- Process changes sau incident: ___
- Bài học rút ra: ___
```

---

## 💡 Câu Hỏi Phỏng Vấn Hay Yêu Cầu STAR Story

Chuẩn bị câu chuyện cho các câu hỏi này:

```
1. "Kể về lần bạn gây ra (hoặc phát hiện) incident production"
   → Câu chuyện 1: State corruption

2. "Describe a time you had to make a critical decision under pressure"
   → Câu chuyện 2: Database deletion

3. "Tell me about a time you improved a process or system"
   → Câu chuyện 3: Drift recovery + new alerting

4. "What's the hardest infrastructure problem you've solved?"
   → Chọn câu chuyện có technical depth nhất

5. "Tell me about a mistake you made and what you learned"
   → Câu chuyện 2 (database) — thể hiện ownership + learning
```

---

## ⚠️ Những Điều Không Nên Làm Khi Kể STAR Story

```
❌ Đổ lỗi cho người khác:
   "Đồng nghiệp tôi đã không review PR kỹ..."
   → Nên: "Team chúng tôi chưa có process review đủ tốt..."

❌ Không có kết quả định lượng:
   "Mọi thứ đã tốt hơn sau đó"
   → Nên: "Downtime giảm từ 2 giờ xuống 0 phút nhờ..."

❌ Quá nhiều technical details, quên human element:
   → Luôn đề cập impact đến users, team, business

❌ Kể chuyện quá hoàn hảo (không có struggle — không có khó khăn):
   → Interviewers muốn thấy bạn handle adversity thật sự

❌ Không có "lesson learned" — bài học rút ra:
   → Luôn kết thúc bằng: "Từ đó, tôi luôn..."
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
