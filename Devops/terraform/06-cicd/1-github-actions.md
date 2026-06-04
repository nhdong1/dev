# GitHub Actions Workflow cho Terraform

> GitHub Actions — Hệ Thống CI/CD Tích Hợp Của GitHub — cho phép tự động hoá quy trình validate, plan, và apply Terraform trực tiếp từ repository.

---

## 🎯 Mục Tiêu

- Hiểu cấu trúc GitHub Actions workflow cho Terraform
- Thiết lập pipeline đầy đủ: validate → lint → security scan → plan → apply
- Cấu hình OIDC — OpenID Connect — thay cho credentials tĩnh
- Xử lý plan output trong Pull Request comment
- Thiết lập approval gate — Cổng phê duyệt — trước khi apply vào production

---

## 🔑 Các Khái Niệm Cơ Bản

### Workflow — Quy Trình Tự Động Hoá

```
Workflow = Tập hợp các jobs được kích hoạt bởi sự kiện (event)

Event (Sự kiện):
├── push: code được đẩy lên
├── pull_request: PR được tạo / cập nhật
├── schedule: chạy theo lịch (cron)
├── workflow_dispatch: kích hoạt thủ công
└── repository_dispatch: kích hoạt qua API

Job (Công việc):
└── Một hoặc nhiều steps chạy trên runner

Step (Bước):
└── Một lệnh shell hoặc action cụ thể
```

### Runner — Máy Chạy CI

```
GitHub-hosted runners (máy do GitHub quản lý):
├── ubuntu-latest    — Linux, miễn phí cho public repos
├── windows-latest   — Windows
└── macos-latest     — macOS

Self-hosted runners (máy tự quản lý):
└── Chạy trong VPC — Virtual Private Cloud — của bạn
    → Phù hợp khi Terraform cần truy cập private network
```

---

## 🏗️ Workflow Cơ Bản — Plan on PR, Apply on Merge

### Cấu Trúc File

```
.github/
└── workflows/
    ├── terraform-plan.yml    # Chạy khi có PR
    └── terraform-apply.yml   # Chạy khi merge vào main
```

### terraform-plan.yml — Chạy Khi Có Pull Request

```yaml
# .github/workflows/terraform-plan.yml
name: "Terraform Plan — Kiểm Tra Thay Đổi"

on:
  pull_request:
    branches:
      - main
    paths:
      - "**.tf"
      - "**.tfvars"

# Quyền cần thiết cho OIDC và PR comments
permissions:
  contents: read
  pull-requests: write
  id-token: write  # Cho OIDC

env:
  TF_VERSION: "1.7.0"
  TF_WORKING_DIR: "./infrastructure"

jobs:
  terraform-plan:
    name: "Terraform Plan"
    runs-on: ubuntu-latest

    steps:
      # Bước 1: Checkout code
      - name: Checkout Repository
        uses: actions/checkout@v4

      # Bước 2: Lấy AWS credentials qua OIDC (không dùng access key tĩnh)
      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1

      # Bước 3: Cài Terraform với version cố định
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      # Bước 4: Cache Terraform providers để tăng tốc
      - name: Cache Terraform Providers
        uses: actions/cache@v4
        with:
          path: ${{ env.TF_WORKING_DIR }}/.terraform
          key: terraform-${{ hashFiles('**/.terraform.lock.hcl') }}

      # Bước 5: terraform init
      - name: Terraform Init
        id: init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init -input=false

      # Bước 6: terraform fmt --check (không sửa, chỉ kiểm tra)
      - name: Terraform Format Check
        id: fmt
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform fmt -check -recursive
        continue-on-error: true  # Báo lỗi nhưng không dừng pipeline

      # Bước 7: terraform validate
      - name: Terraform Validate
        id: validate
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform validate

      # Bước 8: tflint — Kiểm tra best practices
      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: |
          cd ${{ env.TF_WORKING_DIR }}
          tflint --init
          tflint

      # Bước 9: Checkov — Quét bảo mật
      - name: Checkov Security Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ${{ env.TF_WORKING_DIR }}
          framework: terraform
          soft_fail: true  # Không block pipeline, chỉ báo cáo

      # Bước 10: terraform plan và lưu output
      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: |
          terraform plan \
            -input=false \
            -out=tfplan \
            -var-file="environments/dev.tfvars" \
            2>&1 | tee plan_output.txt
        continue-on-error: true

      # Bước 11: Đăng kết quả plan vào PR comment
      - name: Post Plan to PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync(
              '${{ env.TF_WORKING_DIR }}/plan_output.txt', 'utf8'
            );

            // Cắt ngắn nếu quá dài (GitHub có giới hạn comment size)
            const maxLength = 60000;
            const truncated = planOutput.length > maxLength
              ? planOutput.substring(0, maxLength) + '\n...(truncated)'
              : planOutput;

            const output = `## 🔍 Terraform Plan Results

            #### Format: \`${{ steps.fmt.outcome }}\`
            #### Init: \`${{ steps.init.outcome }}\`
            #### Validate: \`${{ steps.validate.outcome }}\`
            #### Plan: \`${{ steps.plan.outcome }}\`

            <details><summary>📋 Chi Tiết Plan</summary>

            \`\`\`hcl
            ${truncated}
            \`\`\`

            </details>

            > Người chạy: @${{ github.actor }}
            > Workflow: ${{ github.workflow }}`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      # Thất bại nếu plan lỗi
      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1
```

### terraform-apply.yml — Chạy Khi Merge Vào Main

```yaml
# .github/workflows/terraform-apply.yml
name: "Terraform Apply — Áp Dụng Thay Đổi"

on:
  push:
    branches:
      - main
    paths:
      - "**.tf"
      - "**.tfvars"

permissions:
  contents: read
  id-token: write

env:
  TF_VERSION: "1.7.0"
  TF_WORKING_DIR: "./infrastructure"

jobs:
  terraform-apply:
    name: "Terraform Apply"
    runs-on: ubuntu-latest
    environment: production  # Kích hoạt manual approval nếu được cấu hình

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init -input=false

      - name: Terraform Plan (Fresh Plan Before Apply)
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: |
          terraform plan \
            -input=false \
            -out=tfplan \
            -var-file="environments/prod.tfvars"

      - name: Terraform Apply
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform apply -input=false -auto-approve tfplan

      # Thông báo kết quả
      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "✅ Terraform Apply thành công trên production\nCommit: ${{ github.sha }}\nBởi: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "❌ Terraform Apply THẤT BẠI trên production\nCommit: ${{ github.sha }}\nBởi: ${{ github.actor }}\nXem logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 🔐 Cấu Hình OIDC — OpenID Connect

### Tại Sao Dùng OIDC Thay Vì Access Key?

```
❌ Cách cũ — Access Key tĩnh:
├── AWS_ACCESS_KEY_ID lưu trong GitHub Secrets
├── Credentials không bao giờ hết hạn
├── Nếu bị lộ → attacker có quyền tồn tại lâu dài
└── Phải rotate định kỳ, dễ bị quên

✅ OIDC — Credentials tạm thời:
├── GitHub Actions nhận JWT — JSON Web Token — từ GitHub
├── Gửi JWT lên AWS STS — Security Token Service
├── AWS kiểm tra và trả về credentials tạm thời (1 giờ)
└── Không có gì để lưu trong Secrets
```

### Thiết Lập OIDC Provider Trên AWS

```hcl
# Tạo OIDC provider trong AWS (chỉ làm một lần)
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # Thumbprint của GitHub's OIDC endpoint
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# IAM Role mà GitHub Actions sẽ assume
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # Chỉ cho phép repo cụ thể
            "token.actions.githubusercontent.com:sub" = "repo:my-org/my-repo:*"
          }
        }
      }
    ]
  })
}

# Đính kèm policy cho role
resource "aws_iam_role_policy_attachment" "github_actions_terraform" {
  role       = aws_iam_role.github_actions.name
  policy_arn = aws_iam_policy.terraform_ci.arn
}
```

### IAM Policy — Quyền Tối Thiểu Cho CI

```hcl
resource "aws_iam_policy" "terraform_ci" {
  name        = "TerraformCIPolicy"
  description = "Quyền tối thiểu cho Terraform CI/CD"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Đọc state từ S3
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          "arn:aws:s3:::my-terraform-state",
          "arn:aws:s3:::my-terraform-state/*"
        ]
      },
      # State locking với DynamoDB
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:ap-southeast-1:*:table/terraform-locks"
      },
      # Quyền tạo/xóa tài nguyên (tùy theo hạ tầng của bạn)
      {
        Effect   = "Allow"
        Action   = ["ec2:*", "ecs:*", "rds:*"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:RequestedRegion" = "ap-southeast-1"
          }
        }
      }
    ]
  })
}
```

---

## 🌍 Workflow Đa Môi Trường — Multi-Environment

### Strategy — Chiến Lược Matrix

```yaml
# .github/workflows/terraform-multi-env.yml
name: "Terraform Multi-Environment"

on:
  push:
    branches: [main, develop]

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - id: set-env
        run: |
          if [ "${{ github.ref }}" = "refs/heads/main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          else
            echo "environment=staging" >> $GITHUB_OUTPUT
          fi

  terraform:
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}

    env:
      ENV_NAME: ${{ needs.determine-environment.outputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          # Role khác nhau cho từng môi trường
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=${{ env.ENV_NAME }}/terraform.tfstate"

      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            -var-file="environments/${{ env.ENV_NAME }}.tfvars"
```

### Cấu Hình Môi Trường Trong GitHub

```
GitHub Repository → Settings → Environments

Tạo environments:
├── staging
│   ├── Required reviewers: không (tự động)
│   └── Secrets: AWS_ROLE_ARN=arn:aws:iam::111:role/StagingRole
│
└── production
    ├── Required reviewers: 2 người phải approve
    ├── Wait timer: 5 phút
    └── Secrets: AWS_ROLE_ARN=arn:aws:iam::222:role/ProductionRole
```

---

## 💰 Tích Hợp Infracost — Ước Tính Chi Phí

```yaml
- name: Setup Infracost
  uses: infracost/actions/setup@v2
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Generate Infracost Cost Estimate
  run: |
    infracost breakdown \
      --path=./infrastructure \
      --format=json \
      --out-file=/tmp/infracost.json

- name: Post Infracost Comment
  uses: infracost/actions/comment@v1
  with:
    path: /tmp/infracost.json
    behavior: update  # Cập nhật comment cũ thay vì tạo mới
```

---

## ⚙️ Workflow Nâng Cao: Saved Plan File

### Tại Sao Phải Lưu Plan File?

```
Vấn đề nếu plan và apply là hai lần chạy khác nhau:

Lần 1 (plan): Tạo EC2 instance t3.micro
              ↓ (15 phút sau, ai đó thay đổi code)
Lần 2 (apply): Apply... nhưng plan nào? Code đã thay đổi!

Giải pháp: Lưu plan file vào artifact và dùng lại
```

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... setup steps ...

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      # Lưu plan file như artifact
      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: terraform-plan-${{ github.sha }}
          path: tfplan
          retention-days: 1  # Chỉ giữ 1 ngày

  apply:
    needs: plan
    runs-on: ubuntu-latest
    environment: production  # Manual approval ở đây

    steps:
      - uses: actions/checkout@v4
      # ... setup steps ...

      # Tải lại plan file đã lưu
      - name: Download Plan Artifact
        uses: actions/download-artifact@v4
        with:
          name: terraform-plan-${{ github.sha }}

      # Apply chính xác plan đã review
      - name: Terraform Apply
        run: terraform apply -input=false tfplan
```

---

## 🐛 Xử Lý Lỗi Thường Gặp

### Lỗi: State Lock

```yaml
- name: Terraform Init with Retry
  run: |
    for i in 1 2 3; do
      terraform init && break
      echo "Retry $i/3..."
      sleep 10
    done

- name: Force Unlock If Needed (Dùng cẩn thận!)
  if: failure()
  run: |
    # Chỉ dùng khi biết chắc không có apply đang chạy
    terraform force-unlock -force ${{ vars.LOCK_ID }}
```

### Lỗi: Provider Version Conflict

```yaml
- name: Clear Terraform Cache
  run: |
    rm -rf .terraform
    rm -f .terraform.lock.hcl

- name: Terraform Init (Fresh)
  run: terraform init -upgrade
```

### Debug Mode — Chế Độ Gỡ Lỗi

```yaml
- name: Terraform Plan (Debug)
  env:
    TF_LOG: DEBUG
    TF_LOG_PATH: /tmp/terraform-debug.log
  run: terraform plan

- name: Upload Debug Log
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: terraform-debug-log
    path: /tmp/terraform-debug.log
```

---

## 📋 Checklist GitHub Actions Terraform

### Cấu Hình Cơ Bản
- [ ] Workflow file trong `.github/workflows/`
- [ ] Terraform version được pin cụ thể
- [ ] OIDC đã được cấu hình (không dùng access key tĩnh)
- [ ] Branch filter: chỉ trigger trên file `.tf` và `.tfvars`

### Bảo Mật
- [ ] IAM role có quyền tối thiểu
- [ ] Secrets lưu trong GitHub Secrets / Environment variables
- [ ] `GITHUB_TOKEN` permissions được giới hạn
- [ ] OIDC condition lọc theo repo cụ thể

### Pipeline Logic
- [ ] Plan chạy trên PR, Apply chạy sau khi merge
- [ ] Plan output được post vào PR comment
- [ ] Manual approval gate cho production environment
- [ ] Saved plan file được dùng lại cho apply

### Quan Sát — Observability
- [ ] Slack / email notification khi apply thất bại
- [ ] Log được lưu đủ để debug
- [ ] Infracost tích hợp để thấy chi phí thay đổi

---

## 🎯 Câu Hỏi Phỏng Vấn Về GitHub Actions + Terraform

**Q: Tại sao không nên dùng `terraform apply -auto-approve` trực tiếp trong CI?**

A: Vì plan output cần được review trước. Nếu apply ngay mà không có ai xem plan, có thể destroy tài nguyên ngoài ý muốn. Best practice là:
1. Chạy plan và lưu vào plan file
2. Đăng plan output vào PR để review
3. Sau khi approve mới chạy apply với plan file đã lưu

**Q: Làm thế nào để nhiều engineer không apply cùng lúc?**

A: Kết hợp nhiều cơ chế:
1. State locking (DynamoDB) — nếu ai đang apply, người khác sẽ bị block
2. Chỉ cho apply từ CI, không cho apply từ máy cá nhân
3. Branch protection: chỉ một PR được merge vào main tại một thời điểm
4. Environment protection rules: serial deployment (không parallel)

**Q: OIDC hoạt động thế nào trong GitHub Actions?**

A: GitHub đóng vai Federated Identity Provider. Khi job chạy, GitHub tạo JWT token chứa thông tin về repo, branch, event. CI gửi JWT này lên AWS STS và yêu cầu AssumeRoleWithWebIdentity. AWS kiểm tra signature của JWT (bằng public key của GitHub) và trả về credentials tạm thời. Credentials này hết hạn sau 1 giờ và chỉ có quyền trong IAM role được cấu hình.

---

**Tiếp Theo:** [2-gitlab-ci.md](2-gitlab-ci.md)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
