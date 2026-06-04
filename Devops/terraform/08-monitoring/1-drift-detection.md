# Drift Detection — Phát Hiện Lệch Cấu Hình Hạ Tầng

> Drift — Lệch cấu hình — xảy ra khi hạ tầng thực tế trên cloud không còn khớp với những gì được mô tả trong Terraform code và state file. Đây là một trong những vấn đề vận hành thường gặp nhất và nguy hiểm nhất trong môi trường production.

---

## 🎯 Drift Là Gì?

```
                    TERRAFORM STATE             THỰC TẾ AWS
                    ─────────────               ──────────
  Security Group:   port 443 open    ≠          port 443 + 8080 open
  EC2 Instance:     t3.medium        ≠          t3.large  (ai đó resize)
  S3 Bucket:        versioning ON    =          versioning ON  (OK)
  IAM Role:         3 policies       ≠          5 policies  (thêm ngoài)
                                     ▲
                                     │
                              ĐÂY LÀ DRIFT
```

**Định nghĩa chính thức:** Drift là sự khác biệt (delta) giữa **desired state** — trạng thái mong muốn — được lưu trong Terraform và **actual state** — trạng thái thực tế — của hạ tầng.

---

## 💣 Tại Sao Drift Nguy Hiểm?

### Kịch Bản Thực Tế 1 — Bảo Mật

```
Tình huống:
  Ops engineer mở port 22 (SSH) trực tiếp trên Console AWS
  để debug khẩn cấp lúc 2 giờ sáng.

  Sau khi debug xong, quên đóng lại.
  Terraform state vẫn nói port 22 đóng.
  Security scanner dựa vào Terraform → không phát hiện.

Hậu quả:
  - Port 22 public mở trong 3 tuần
  - Không ai biết cho đến khi có audit bảo mật
```

### Kịch Bản Thực Tế 2 — Chi Phí

```
Tình huống:
  Team data tạo thủ công 20 RDS read replicas để test hiệu năng.
  Quên xóa sau khi test xong.
  Terraform không biết những instances này tồn tại.

Hậu quả:
  - $8,000/tháng chi phí không giải thích được
  - Mất 2 tuần để truy ra nguyên nhân
```

### Kịch Bản Thực Tế 3 — Tính Nhất Quán

```
Tình huống:
  terraform apply lần sau ghi đè lên thay đổi thủ công.
  Load balancer health check bị đổi lại → traffic routing sai.

Hậu quả:
  - Outage 40 phút vào giờ cao điểm
  - Không rõ nguyên nhân vì "code không thay đổi"
```

---

## 🔍 Các Loại Drift

### 1. Out-of-Band Change — Thay Đổi Ngoài Luồng

Thay đổi trực tiếp qua Console, CLI, hoặc API mà không qua Terraform.

```bash
# Ai đó chạy AWS CLI trực tiếp
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456 \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0
# → Drift: Terraform không biết rule này tồn tại
```

### 2. Resource Deletion Drift — Tài Nguyên Bị Xóa Ngoài

```bash
# Ai đó xóa S3 bucket trực tiếp
aws s3 rb s3://my-terraform-bucket --force
# → Terraform nghĩ bucket vẫn tồn tại
# → terraform apply sẽ cố recreate → có thể gây lỗi
```

### 3. Provider-Managed Drift — Drift Do Provider

Một số thay đổi xảy ra tự động do cloud provider:

```hcl
# AWS tự động thêm default rules vào Security Group
# AWS tự động rotate credentials
# GCP tự động cập nhật cluster version
# → Không phải lỗi của ai, nhưng vẫn là drift
```

### 4. Terraform-Managed Drift — Drift Do Terraform Tự Gây

```hcl
# Provider version update thay đổi default values
# Terraform version mới tính toán khác
resource "aws_instance" "web" {
  # monitoring mặc định false trong v3.x
  # monitoring mặc định true trong v4.x → drift!
}
```

---

## 🛠️ Phát Hiện Drift

### Phương Pháp 1 — `terraform plan` Định Kỳ (Cơ Bản)

Cách đơn giản nhất: chạy `terraform plan` và kiểm tra output có thay đổi gì không.

```bash
# Chạy plan và lưu exit code
terraform plan -detailed-exitcode
# Exit codes:
#   0 = Không thay đổi (no drift)
#   1 = Lỗi
#   2 = Có thay đổi (drift detected!)
```

**Giải thích flag `-detailed-exitcode`:**
- Exit code `2` có nghĩa plan thành công VÀ có changes → đây là drift nếu không ai sửa code.

### Phương Pháp 2 — `terraform refresh` + `terraform show`

```bash
# Cập nhật state từ thực tế cloud
terraform refresh

# So sánh state trước và sau
terraform show -json > current-state.json

# Diff với state đã commit
git diff terraform.tfstate
```

> **Lưu ý:** `terraform refresh` từ Terraform v0.15.4 đã deprecated. Thay thế bằng `terraform apply -refresh-only`.

```bash
# Cách hiện đại — chỉ refresh không apply
terraform apply -refresh-only

# Với auto-approve để dùng trong automation
terraform apply -refresh-only -auto-approve
```

### Phương Pháp 3 — Driftctl (Công Cụ Chuyên Biệt)

[Driftctl](https://driftctl.com/) là công cụ open-source chuyên phát hiện drift, kể cả tài nguyên **không được quản lý bởi Terraform**.

```bash
# Cài đặt Driftctl
brew install driftctl  # macOS
# hoặc
curl https://driftctl.com/install | bash  # Linux

# Scan toàn bộ AWS account
driftctl scan

# Output mẫu:
# Found 3 drifted resources:
#   aws_security_group.web: 1 change
#     + ingress rule: port 8080
#   aws_iam_role.lambda: 2 changes
#     + policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
# 
# Found 5 unmanaged resources:
#   aws_s3_bucket.manual-bucket-1
#   aws_ec2_instance.test-instance (created 3 weeks ago)
```

**Điểm mạnh của Driftctl:**
- Phát hiện **unmanaged resources** — tài nguyên tồn tại nhưng không trong Terraform
- Hỗ trợ nhiều cloud providers
- Output dạng JSON có thể parse tự động

### Phương Pháp 4 — Terraform Cloud Drift Detection

Terraform Cloud (HCP Terraform) có tính năng drift detection tích hợp:

```
Cấu hình trong Terraform Cloud:
  Workspace Settings → Health → Drift Detection
  → Chọn frequency: Hourly / Daily
  → Nhận notification khi phát hiện drift
```

---

## 🤖 Tự Động Hóa Drift Detection Trong CI/CD

### GitHub Actions — Scheduled Drift Check

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection — Phát Hiện Lệch Cấu Hình

on:
  schedule:
    # Chạy mỗi ngày lúc 8 giờ sáng UTC (3 giờ chiều VN)
    - cron: '0 8 * * *'
  workflow_dispatch:  # Cho phép chạy thủ công

jobs:
  drift-check:
    name: Check for Infrastructure Drift
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1

      - name: Terraform Init
        run: terraform init
        working-directory: ./infrastructure

      - name: Check for Drift
        id: drift
        run: |
          # -detailed-exitcode: exit 2 nếu có changes
          terraform plan -detailed-exitcode -no-color 2>&1 | tee plan-output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        working-directory: ./infrastructure
        continue-on-error: true  # Không fail job khi có drift

      - name: Parse Drift Results
        run: |
          EXIT_CODE=${{ steps.drift.outputs.exit_code }}
          if [ "$EXIT_CODE" -eq "2" ]; then
            echo "⚠️ DRIFT DETECTED — Lệch cấu hình phát hiện!"
            grep -E "^  [+~-]" plan-output.txt || true
          elif [ "$EXIT_CODE" -eq "0" ]; then
            echo "✅ No drift detected — Không có lệch cấu hình"
          else
            echo "❌ Error running terraform plan"
            exit 1
          fi

      - name: Send Slack Alert if Drift Found
        if: steps.drift.outputs.exit_code == '2'
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              "text": "⚠️ *Drift Detected* — Phát hiện lệch cấu hình hạ tầng!",
              "attachments": [{
                "color": "warning",
                "fields": [
                  {
                    "title": "Repository",
                    "value": "${{ github.repository }}",
                    "short": true
                  },
                  {
                    "title": "Branch",
                    "value": "${{ github.ref_name }}",
                    "short": true
                  },
                  {
                    "title": "Action",
                    "value": "Review và remediate ngay: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Upload Plan Output
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: drift-plan-output
          path: infrastructure/plan-output.txt
          retention-days: 30
```

### GitLab CI — Drift Detection Pipeline

```yaml
# .gitlab-ci.yml (thêm vào pipeline hiện có)
drift-detection:
  stage: monitoring
  image: hashicorp/terraform:1.7
  rules:
    # Chạy theo lịch (cấu hình trong GitLab Schedules)
    - if: $CI_PIPELINE_SOURCE == "schedule"
    # Hoặc chạy thủ công
    - if: $CI_PIPELINE_SOURCE == "web"
  
  before_script:
    - terraform init
  
  script:
    - |
      terraform plan -detailed-exitcode -no-color > plan-output.txt 2>&1
      EXIT_CODE=$?
      
      if [ $EXIT_CODE -eq 2 ]; then
        echo "DRIFT_FOUND=true" >> drift.env
        echo "⚠️ Drift detected in $(pwd)"
        cat plan-output.txt
      elif [ $EXIT_CODE -eq 0 ]; then
        echo "DRIFT_FOUND=false" >> drift.env
        echo "✅ No drift"
      fi
  
  artifacts:
    reports:
      dotenv: drift.env
    paths:
      - plan-output.txt
    expire_in: 30 days

notify-drift:
  stage: notify
  needs:
    - job: drift-detection
      artifacts: true
  rules:
    - if: $DRIFT_FOUND == "true"
  script:
    - |
      curl -X POST $SLACK_WEBHOOK \
        -H 'Content-type: application/json' \
        --data '{"text":"⚠️ Drift detected in '"$CI_PROJECT_NAME"'"}'
```

---

## 🔧 Xử Lý Drift — Remediation

Khi phát hiện drift, có 3 hướng xử lý:

### Hướng 1 — Re-apply Terraform (Ghi đè thay đổi thủ công)

```bash
# Terraform sẽ đưa hạ tầng về đúng state trong code
terraform apply

# Thích hợp khi:
# - Thay đổi thủ công là sai/không được phép
# - Muốn enforce "code là nguồn sự thật duy nhất"
```

### Hướng 2 — Update Code Để Match Thực Tế

```bash
# 1. Xem thay đổi chi tiết
terraform plan -out=drift.tfplan
terraform show -json drift.tfplan

# 2. Cập nhật code để phản ánh thay đổi mong muốn
# Ví dụ: thêm rule 8080 vào security group trong code

# 3. Apply lại
terraform apply
```

**Thích hợp khi:**
- Thay đổi thủ công là đúng và nên giữ lại
- Muốn "adopt" thay đổi vào IaC — Infrastructure as Code

### Hướng 3 — Import Tài Nguyên Không Được Quản Lý

```bash
# Tài nguyên tồn tại nhưng Terraform không biết
terraform import aws_s3_bucket.manual_bucket my-manual-bucket-name

# Sau đó viết code cho tài nguyên này
# terraform plan sẽ không thấy diff nữa
```

### Quyết Định Hướng Xử Lý

```
Phát hiện drift
      │
      ▼
Drift có chủ ý?
(ai đó muốn giữ thay đổi)
      │
   ┌──┴──┐
  Có    Không
   │      │
   ▼      ▼
Update  Re-apply
 code  Terraform
   │      │
   └──┬───┘
      │
      ▼
 PR review
 & merge
      │
      ▼
 Drift resolved ✅
```

---

## 📊 Metrics — Chỉ Số Theo Dõi Drift

| Chỉ Số | Mô Tả | Mục Tiêu |
|--------|--------|----------|
| Drift Detection Rate | Tần suất phát hiện drift | Hàng ngày hoặc hàng giờ |
| Time to Detection (TTD) | Thời gian từ khi drift xảy ra đến khi phát hiện | < 24 giờ |
| Time to Remediation (TTR) | Thời gian từ khi phát hiện đến khi xử lý | < 4 giờ (P1) |
| Drift Frequency | Số lần drift/tuần | Giảm dần theo thời gian |
| Unmanaged Resources | Số tài nguyên không qua Terraform | → 0 |

---

## 🚫 Anti-Patterns — Những Lỗi Thường Gặp

### 1. Bỏ Qua Drift Thường Xuyên

```
❌ Sai: "Drift nhỏ, không quan trọng, để sau giải quyết"
✅ Đúng: Mỗi drift dù nhỏ đều phải được review và resolved
         Drift nhỏ hôm nay → incident lớn tuần sau
```

### 2. Chỉ Dùng `terraform refresh` Mà Không Review

```bash
# ❌ Sai: Tự động refresh mà không xem thay đổi là gì
terraform apply -refresh-only -auto-approve  # Nguy hiểm!

# ✅ Đúng: Luôn review trước khi refresh
terraform apply -refresh-only  # Terraform hỏi confirm
```

### 3. Không Có Drift Detection Trong Production

```
❌ Sai: Chỉ có drift detection trong dev/staging
✅ Đúng: Production cần drift detection NGHIÊM NGẶT hơn
         vì thay đổi ngoài luồng thường xảy ra ở production
         khi incident response khẩn cấp
```

### 4. Không Ghi Lại Lý Do Thay Đổi Ngoài Luồng

```
❌ Sai: Thay đổi thủ công không có documentation
✅ Đúng: Luôn tạo ticket/issue ghi lại:
         - Ai thay đổi
         - Tại sao thay đổi thủ công (không qua Terraform)
         - Kế hoạch bring vào IaC khi nào
```

---

## 🎯 Best Practices — Thực Hành Tốt Nhất

1. **Scheduled drift detection hàng ngày** — Đặt lịch chạy `terraform plan` vào giờ cố định, alert nếu có changes.

2. **Drift detection trước mỗi `terraform apply`** — Luôn chạy `terraform plan` trước apply để thấy có drift không.

3. **Immutable infrastructure** — Với EC2/VM, thay vì sửa instance đang chạy, hãy terminate và tạo mới qua Terraform.

4. **Restrict console access** — Dùng IAM permissions để giới hạn ai được phép thay đổi trực tiếp, buộc mọi thay đổi qua Terraform.

5. **Drift remediation SLA** — Đặt Service Level Agreement — Thỏa thuận mức dịch vụ — cho việc xử lý drift: P0 (security) < 1 giờ, P1 < 4 giờ, P2 < 24 giờ.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Drift detection là gì? Bạn xử lý drift như thế nào trong team?**

> Drift là sự khác biệt giữa trạng thái mong muốn trong Terraform code/state và trạng thái thực tế của hạ tầng. Tôi xử lý bằng cách: (1) chạy scheduled `terraform plan` hàng ngày với `-detailed-exitcode`, (2) alert qua Slack khi phát hiện drift, (3) triage và quyết định xử lý theo hướng nào tùy theo context — re-apply hoặc update code. Quan trọng là phải document lý do mỗi quyết định.

**Q: Làm thế nào để ngăn chặn drift thay vì chỉ phát hiện?**

> Prevention tốt hơn detection: (1) Restrict IAM permissions — không ai được thay đổi production trực tiếp ngoại trừ Terraform CI/CD role; (2) Enforce change management process — mọi thay đổi phải qua PR; (3) Use immutable infrastructure patterns; (4) Dùng AWS Config rules hoặc OPA policies để auto-remediate một số loại drift.

**Q: Trong incident response, kỹ sư thường cần thay đổi hạ tầng nhanh mà không có thời gian làm PR. Bạn xử lý thế nào?**

> Đây là conflict thực tế giữa speed và process. Giải pháp của tôi: (1) Có "break glass" procedure cho P0 — cho phép thay đổi thủ công khẩn cấp; (2) Bắt buộc tạo ticket ngay khi dùng break glass; (3) Bắt buộc bring change vào IaC trong 24h sau khi incident resolve; (4) Post-mortem review để cải thiện quy trình. Điều quan trọng là không để drift tồn tại lâu dài.

---

## 🔗 Tài Liệu Tham Khảo

- [Terraform: Detect and Manage Drift](https://developer.hashicorp.com/terraform/tutorials/state/resource-drift)
- [Driftctl Documentation](https://driftctl.com/docs/)
- [Terraform Cloud Drift Detection](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/health)

---

**Tiếp Theo:** [2-infracost.md](./2-infracost.md) — Ước tính chi phí trong CI/CD

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
