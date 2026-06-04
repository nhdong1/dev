# Thiết Kế Hệ Thống với IaC — Infrastructure as Code

> Phần này hướng dẫn cách tiếp cận bài toán System Design — Thiết Kế Hệ Thống — có yếu tố Infrastructure as Code. Kèm framework trả lời, câu hỏi mẫu, và diagram mẫu.

---

## 🎯 Framework Trả Lời System Design với IaC

Dùng framework **CADR** — Constraints, Architecture, Design, Risks:

```
1. C — Constraints (Ràng buộc):
   - Team size — Quy mô team? (1 người vs 50 người)
   - Cloud provider? (AWS / GCP / Azure / Multi-cloud)
   - Compliance requirements? (PCI-DSS, HIPAA, SOC2)
   - Existing infrastructure có không?
   - Budget constraints?

2. A — Architecture (Kiến trúc tổng thể):
   - Bao nhiêu environments? (dev/staging/prod)
   - Single region hay multi-region?
   - Shared infrastructure hay isolated per team?

3. D — Design (Thiết kế chi tiết):
   - Module structure — Cấu trúc module
   - State management strategy — Chiến lược quản lý state
   - CI/CD workflow — Quy trình CI/CD
   - Security controls — Kiểm soát bảo mật

4. R — Risks & Mitigations (Rủi ro & Giải pháp):
   - What can go wrong? — Điều gì có thể sai?
   - How to recover? — Cách khôi phục?
   - Trade-offs — Đánh đổi giữa các lựa chọn
```

---

## 🏗️ Bài Toán 1: Thiết Kế IaC cho Startup (20 người, AWS)

### Đề Bài

> "Bạn join một startup 20 engineers. Hiện tại mọi thứ được setup manual qua AWS Console. CEO muốn 'infrastructure as code'. Hãy thiết kế approach từ đầu."

### Phân Tích Yêu Cầu

```
Constraints:
- Small team: không cần phức tạp quá
- Greenfield: có thể design từ đầu (không phải migrate)
- Budget sensitive: minimize tool costs
- Speed matters: startup cần ship nhanh
- 3 environments: dev, staging, production

Requirements:
- Reproducible environments — Môi trường có thể tái tạo
- Safe production deployments — Triển khai production an toàn
- Secret management — Quản lý bí mật
- Cost visibility — Hiển thị chi phí
```

### Architecture Design

```
Repository Structure — Cấu trúc Repository:

infrastructure/
├── modules/                    # Reusable modules
│   ├── networking/
│   │   ├── main.tf            # VPC, subnets, NAT Gateway
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf            # ECS cluster, ALB
│   │   └── ...
│   ├── database/
│   │   ├── main.tf            # RDS, ElastiCache
│   │   └── ...
│   └── security/
│       ├── main.tf            # IAM roles, Security Groups
│       └── ...
│
├── environments/
│   ├── dev/
│   │   ├── main.tf            # Module calls
│   │   ├── variables.tf
│   │   ├── terraform.tfvars   # dev-specific values
│   │   └── backend.tf         # S3 backend config
│   ├── staging/
│   └── production/
│
├── global/                    # Cross-env resources
│   ├── iam/                   # IAM users, groups
│   ├── route53/               # DNS zones
│   └── ecr/                   # Container registry
│
└── scripts/
    ├── bootstrap.sh           # Setup backend S3 + DynamoDB
    └── plan-all.sh            # Plan tất cả environments
```

### State Management — Quản Lý Trạng Thái

```hcl
# environments/production/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/xxx"
  }
}

# S3 bucket setup (bootstrap — khởi tạo ban đầu):
# - Versioning: enabled (cho phép rollback state)
# - Server-side encryption: AES-256 hoặc KMS
# - Block public access: enabled
# - Replication: optional (cho DR — Disaster Recovery)
```

### CI/CD Workflow

```yaml
# .github/workflows/terraform.yml (GitHub Actions)
# PR workflow:
on:
  pull_request:
    paths: ['infrastructure/**']

jobs:
  plan:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/TerraformPlanRole
          aws-region: us-east-1

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Plan
        run: |
          cd infrastructure/environments/${{ matrix.env }}
          terraform init
          terraform plan -out=plan.tfplan
          terraform show -no-color plan.tfplan > plan.txt

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              body: '```\n' + require('fs').readFileSync('plan.txt') + '\n```'
            })

# Merge to main → apply (sau khi approve)
```

### Security Controls

```
IAM Strategy — Chiến lược IAM:
- CI/CD Plan role: ReadOnly permissions cho mọi services
- CI/CD Apply role: Write permissions với resource tagging conditions
- Never: AdministratorAccess trong CI/CD
- Use OIDC — OpenID Connect — thay vì access keys

Secrets Strategy — Chiến lược Bí mật:
- AWS SSM Parameter Store cho app configs
- AWS Secrets Manager cho database passwords
- Terraform variables chỉ chứa non-sensitive values
- NEVER: secrets trong .tfvars files trong git

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Trade-offs & Alternatives

```
Lựa chọn này (Directory per env) vs Terraform Workspaces:
+ Đơn giản hơn, dễ hiểu cho team mới
+ Mỗi env có state riêng biệt hoàn toàn
+ Dễ có different configs per env
- Code lặp nhiều hơn
- Cần Terragrunt nếu scale lên nhiều services

Lựa chọn này vs Terraform Cloud:
+ Free (chỉ tốn tiền S3 + DynamoDB)
+ Full control
- Phải tự setup và maintain
- Không có built-in UI cho state history
→ Startup early-stage: Self-managed trên S3 OK
→ Khi team > 10 người DevOps: Consider Terraform Cloud
```

---

## 🏗️ Bài Toán 2: Multi-Account AWS Architecture — Kiến Trúc Đa Tài Khoản

### Đề Bài

> "Công ty fintech, 200 engineers, 50 microservices, compliance PCI-DSS. Thiết kế AWS multi-account strategy và Terraform structure tương ứng."

### Phân Tích

```
Constraints fintech/PCI-DSS:
- Separation of duties — Phân tách nhiệm vụ: không ai có full access
- Audit trail: mọi action phải được log
- Network isolation: cardholder data environment phải isolated
- Least privilege: IAM permissions phải minimal

Scale considerations:
- 50 microservices = nhiều state files
- 200 engineers = nhiều teams, cần self-service
- Multiple compliance scopes (PCI, non-PCI)
```

### Multi-Account Structure

```
AWS Organizations:
├── Management Account (000000000000)
│   ├── Billing consolidation
│   ├── SCPs — Service Control Policies — enforcement
│   └── Cross-account roles
│
├── Security Account (111111111111)
│   ├── CloudTrail logs (tất cả accounts)
│   ├── Security Hub aggregation
│   ├── GuardDuty master
│   └── Terraform state cho security configs
│
├── Shared Services Account (222222222222)
│   ├── Shared VPC Transit Gateway
│   ├── Container registry (ECR)
│   ├── Internal tools (Atlantis, monitoring)
│   └── DNS (Route53 private zones)
│
├── Production Accounts (per domain):
│   ├── payments-prod (333333333333) — PCI scope
│   ├── platform-prod (444444444444)
│   └── data-prod (555555555555)
│
└── Non-Production Accounts:
    ├── dev (666666666666)
    └── staging (777777777777)
```

### Terraform Structure cho Multi-Account

```
infrastructure/
├── account-bootstrap/        # Setup mỗi account lần đầu
│   ├── main.tf               # S3 bucket, DynamoDB, IAM roles
│   └── README.md
│
├── modules/                  # Shared modules library
│   ├── networking/
│   ├── compute/
│   ├── security/
│   └── observability/
│
├── live/                     # Actual deployments
│   ├── payments-prod/
│   │   ├── us-east-1/
│   │   │   ├── vpc/
│   │   │   │   └── terragrunt.hcl
│   │   │   ├── eks/
│   │   │   └── rds/
│   │   └── us-west-2/        # DR region
│   ├── platform-prod/
│   └── dev/
│
└── terragrunt.hcl            # Root config: backend, providers
```

### Cross-Account Access Pattern

```hcl
# Module accounts/cross-account-role/main.tf
# Trust policy cho CI/CD từ Shared Services account
data "aws_iam_policy_document" "assume_role" {
  statement {
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${var.cicd_account_id}:role/TerraformExecutor"]
    }
    actions = ["sts:AssumeRole"]
    condition {
      test     = "StringEquals"
      variable = "sts:ExternalId"
      values   = [var.external_id]
    }
  }
}

resource "aws_iam_role" "terraform_deploy" {
  name               = "TerraformDeploy-${var.environment}"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

# Provider config với cross-account assume role
provider "aws" {
  alias  = "payments_prod"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformDeploy-production"
    session_name = "terraform-cicd"
  }
}
```

### State Management cho Multi-Account

```
State isolation strategy — Chiến lược cô lập state:

Option A — State per account per service (khuyến nghị):
s3://company-terraform-state-{account_id}/
  ├── us-east-1/vpc/terraform.tfstate
  ├── us-east-1/eks/terraform.tfstate
  └── us-east-1/rds/terraform.tfstate

Ưu điểm:
- Blast radius — Phạm vi ảnh hưởng — nhỏ khi có sự cố
- Clear ownership per team
- Easy access control per state file
- Concurrent operations không conflict

Nhược điểm:
- Nhiều state files cần manage
- Cross-state dependencies phức tạp hơn
  (dùng terraform_remote_state data source)

Chia sẻ outputs giữa state files:
# Trong payments-prod/eks/main.tf
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "company-terraform-state-333333333333"
    key    = "us-east-1/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
  }
}
```

---

## 🏗️ Bài Toán 3: Zero-Downtime Migration — Chuyển Đổi Không Gián Đoạn

### Đề Bài

> "Công ty có 1000 resources được tạo manual trên AWS. Bạn cần migrate tất cả vào Terraform mà không có downtime. Approach của bạn?"

### Phân Tích Risk

```
Risks — Rủi ro:
1. Import sai resource config → apply sẽ modify/delete resources
2. Missing dependencies → apply thứ tự sai
3. Unknown configurations → thay đổi ngoài ý muốn
4. Team chưa biết Terraform → human error

Principles — Nguyên tắc:
- Incremental migration — Chuyển đổi dần dần (không big bang)
- Verify at every step — Kiểm tra sau mỗi bước
- Maintain rollback capability — Giữ khả năng rollback
- Team education first — Training trước
```

### Migration Plan — Kế Hoạch Chuyển Đổi

```
Phase 1 — Foundation (Tuần 1-2):
  1. Setup Terraform environment (remote backend, CI/CD)
  2. Train team: 2-hour workshop về Terraform basics
  3. Create module library từ existing patterns
  4. Start với non-critical resources (S3 buckets, IAM roles)

Phase 2 — Import (Tuần 3-6):
  Group by service:
  a. Networking layer: VPC, Subnets, Route Tables, Security Groups
  b. IAM: Roles, Policies, Instance Profiles
  c. Database: RDS, ElastiCache (stateful — cẩn thận nhất)
  d. Compute: EC2, Auto Scaling Groups
  e. Application layer: ECS Services, Load Balancers

Phase 3 — Validation (Song song với Phase 2):
  Sau mỗi import batch:
  - terraform plan phải thấy "No changes" (zero drift)
  - Integration tests vẫn pass
  - Monitor application metrics (error rate, latency)

Phase 4 — Lock Down (Tuần 7-8):
  - Bật AWS Config Rules: alert khi manual changes
  - Remove Console access cho resources under Terraform
  - Setup drift detection cron job
```

### Import Workflow — Quy Trình Import

```bash
# Script để import một resource và verify
import_and_verify() {
  local resource_address=$1
  local resource_id=$2

  echo "=== Importing $resource_address ==="

  # Step 1: Import
  terraform import "$resource_address" "$resource_id"

  # Step 2: Show what was imported
  terraform show -json | jq ".values.root_module.resources[] |
    select(.address == \"$resource_address\")"

  # Step 3: Plan — must be zero changes
  terraform plan -target="$resource_address" \
    -detailed-exitcode
  local exit_code=$?

  if [ $exit_code -eq 0 ]; then
    echo "✅ Import successful, no drift"
  elif [ $exit_code -eq 2 ]; then
    echo "⚠️  Import done but drift detected — update code to match"
    echo "    Run: terraform show to see actual configuration"
  else
    echo "❌ Error during plan — investigate"
    exit 1
  fi
}

# Example usage:
import_and_verify "aws_vpc.main" "vpc-0abc123"
import_and_verify "aws_security_group.web" "sg-0def456"
```

### Terraform 1.5+ Import Blocks — Khai Báo Import

```hcl
# Terraform 1.5+ cho phép import declarative
# Tốt hơn cho batch imports và GitOps workflow

import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789a"
}

import {
  to = aws_security_group.web
  id = "sg-0abc123def456789a"
}

# Generate config tự động (Terraform 1.5+)
# terraform plan -generate-config-out=imported_resources.tf
# Terraform sẽ tự tạo resource blocks với đúng values
```

---

## 📊 Comparison Table — Bảng So Sánh Approaches

| Kịch Bản | Approach | State Strategy | CI/CD Tool |
|----------|----------|----------------|------------|
| Startup < 10 người | Directory per env | S3 + DynamoDB | GitHub Actions |
| Startup > 20 người | Terragrunt | S3 per env | GitHub Actions + Atlantis |
| Enterprise multi-team | Terragrunt + modules registry | Per account per service | Terraform Cloud hoặc Atlantis |
| Multi-cloud | Workspace hoặc separate repos | Per cloud per env | GitLab CI |
| Compliance-heavy | Separate accounts + Sentinel | Per account, encrypted KMS | Terraform Enterprise |

---

## 🎤 Tips Trả Lời System Design Interview

```
1. Clarify first — Hỏi rõ trước:
   "Trước khi design, cho tôi hỏi thêm:
   - Team size? Single team hay nhiều teams?
   - Compliance requirements nào? (PCI, HIPAA, SOC2)
   - Greenfield hay existing infrastructure?
   - Budget constraint?"

2. Think aloud — Nói to suy nghĩ:
   "Tôi thấy có 2 approaches:
   Option A: đơn giản hơn nhưng không scale tốt khi...
   Option B: phức tạp hơn nhưng sẽ cần khi...
   Với constraints bạn đưa ra, tôi sẽ chọn A vì..."

3. Draw diagram — Vẽ sơ đồ:
   Luôn vẽ hoặc mô tả cấu trúc thư mục, flow của CI/CD,
   data flow trong state management

4. Address operability — Đề cập vận hành:
   "Design này hoạt động tốt vì... nhưng team sẽ cần
   biết/làm... khi... xảy ra"

5. Quantify — Định lượng:
   "Approach này sẽ giảm deployment time từ 2 giờ manual
   xuống 15 phút automated, và loại bỏ human error trong..."
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
