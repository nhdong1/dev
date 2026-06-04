# Kế Hoạch Học Terraform 90 Ngày

> Lộ trình học có cấu trúc từ zero đến sẵn sàng phỏng vấn DevOps Engineer / Platform Engineer. Mỗi tuần có mục tiêu rõ ràng, bài tập thực hành và checklist tự đánh giá.

---

## 🎯 Tổng Quan 90 Ngày

```
Tháng 1 (Ngày 1-30):   Nền tảng vững chắc
Tháng 2 (Ngày 31-60):  Kỹ năng vận hành
Tháng 3 (Ngày 61-90):  Chuyên sâu & chuẩn bị phỏng vấn
```

### Mục Tiêu Cuối Chương Trình

Sau 90 ngày, bạn có thể:
- ✅ Thiết kế và triển khai infrastructure phức tạp với Terraform
- ✅ Tích hợp Terraform vào CI/CD pipeline thực tế
- ✅ Tự tin trả lời mọi câu hỏi phỏng vấn Terraform
- ✅ Kể được 3+ câu chuyện STAR về kinh nghiệm hạ tầng
- ✅ Đủ điều kiện apply vào vị trí DevOps Engineer / Platform Engineer

---

## 📅 Tháng 1: Nền Tảng (Ngày 1–30)

### Tuần 1 (Ngày 1–7): Terraform Fundamentals — Nền Tảng

**Mục tiêu tuần:**
- Hiểu IaC là gì và tại sao Terraform
- Viết và chạy được Terraform code đầu tiên
- Tạo resource thực tế trên AWS/GCP

**Ngày 1-2: Setup & Hello World**

```bash
# 1. Cài đặt tools
brew install terraform tflint    # macOS
# Windows: winget install Hashicorp.Terraform
terraform version   # Verify >= 1.5.0

# 2. Cài AWS CLI và cấu hình credentials
aws configure
# Dùng AWS Free Tier — không tốn tiền cho resources nhỏ

# 3. Terraform đầu tiên
mkdir ~/terraform-lab && cd ~/terraform-lab

cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "learning" {
  bucket = "my-terraform-learning-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}
EOF

terraform init
terraform plan
terraform apply
terraform destroy   # QUAN TRỌNG: Destroy để không tốn tiền
```

**Ngày 3-4: HCL Syntax — Cú Pháp HCL**
- Học: Variables, locals, outputs, data sources
- Đọc: `01-fundamentals/2-hcl-syntax.md`
- Lab: Tạo EC2 instance với variable cho instance_type

**Ngày 5-6: Providers & Resources**
- Đọc: `01-fundamentals/3-providers-resources.md`
- Lab: Tạo VPC + Subnet + EC2 instance kết nối nhau

**Ngày 7: Review & Consolidate — Ôn Tập**
- Review code đã viết tuần này
- Fix bất kỳ issues nào phát hiện
- Viết notes về những gì chưa hiểu

**Checklist Tuần 1:**
```
[ ] terraform init/plan/apply/destroy chạy thành công
[ ] Hiểu state file lưu ở đâu và chứa gì
[ ] Viết variable với type, description, validation
[ ] Dùng output để lấy resource attributes
[ ] Tạo ít nhất 5 resources khác nhau trên AWS
```

---

### Tuần 2 (Ngày 8–14): State Management — Quản Lý Trạng Thái

**Mục tiêu tuần:**
- Setup remote backend hoạt động
- Hiểu state locking và tại sao cần
- Biết cách xử lý state operations cơ bản

**Ngày 8-9: Remote Backend Setup**
```hcl
# Lab: Setup S3 + DynamoDB backend
# bootstrap/main.tf

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state-${var.account_id}"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Ngày 10-11: State Commands**
```bash
# Lab: Thực hành state commands
terraform state list
terraform state show aws_s3_bucket.my_bucket
terraform state pull > backup.tfstate
terraform state mv aws_s3_bucket.old aws_s3_bucket.new
terraform state rm aws_s3_bucket.temporary  # (không destroy)
```

**Ngày 12-13: Simulated Corruption Recovery — Mô Phỏng Khôi Phục**
```bash
# Lab: Tự tay "phá" state (trong môi trường test!)
# Backup state
terraform state pull > good-state.tfstate

# Mô phỏng corruption: xóa một resource khỏi state
terraform state rm aws_instance.web

# Verify drift
terraform plan  # Sẽ muốn tạo lại instance đã bị xóa khỏi state

# Recovery: import lại
terraform import aws_instance.web i-1234567890abcdef0

# Verify
terraform plan  # Phải thấy "No changes"
```

**Ngày 14: Review**
- Đọc: `02-state-management/5-state-recovery.md`
- Viết STAR story đầu tiên về state management

**Checklist Tuần 2:**
```
[ ] Remote backend S3 + DynamoDB đang hoạt động
[ ] Chạy 5 state commands khác nhau
[ ] Simulate và recover từ state corruption
[ ] Giải thích được state locking mechanism
[ ] Setup S3 versioning và verify backup còn đó
```

---

### Tuần 3 (Ngày 15–21): Modules — Mô-đun

**Mục tiêu tuần:**
- Viết module tái sử dụng đầu tiên
- Hiểu module versioning và registry
- Dùng official modules từ Terraform Registry

**Ngày 15-16: First Module**
```hcl
# Lab: Viết VPC module
# modules/vpc/main.tf
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "environment" {
  type = string
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

# root/main.tf - gọi module
module "vpc" {
  source      = "./modules/vpc"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}
```

**Ngày 17-18: Official Modules**
```hcl
# Lab: Dùng terraform-aws-modules/vpc
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true  # Cost-saving cho dev

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

**Ngày 19-20: Module Design Principles**
- Đọc: `03-modules/5-composition-patterns.md`
- Lab: Refactor tuần trước thành proper modules với:
  - Sensible defaults
  - Input validation
  - Meaningful outputs
  - README.md

**Ngày 21: Review & Mini-Project**
- Tạo mini-project: VPC + ECS cluster từ modules
- Verify với `terraform plan` ra "No changes"

**Checklist Tuần 3:**
```
[ ] Viết module với variables, locals, outputs
[ ] Module có README.md và examples
[ ] Dùng ít nhất 2 official modules từ Registry
[ ] Understand semantic versioning cho modules
[ ] Module có input validation với error messages
```

---

### Tuần 4 (Ngày 22–30): Environments & Security Basics

**Mục tiêu tuần:**
- Setup dev/staging/prod separation
- Implement secrets management cơ bản
- Hiểu IAM least privilege cho Terraform

**Ngày 22-24: Multi-Environment Setup**
```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── production/
    ├── main.tf
    └── terraform.tfvars

Lab: Deploy cùng module ra 3 envs với config khác nhau
Verify: 3 state files riêng biệt
```

**Ngày 25-27: Secrets Management**
```hcl
# Lab: AWS SSM Parameter Store
resource "aws_ssm_parameter" "db_password" {
  name  = "/myapp/production/db_password"
  type  = "SecureString"
  value = var.db_password
}

# Read trong application:
data "aws_ssm_parameter" "db_password" {
  name = "/myapp/production/db_password"
}

# Lab: Sensitive variables
variable "api_key" {
  type      = string
  sensitive = true
}

output "api_endpoint" {
  value     = "https://api.company.com"
  sensitive = false  # OK to show
}

output "api_key" {
  value     = var.api_key
  sensitive = true   # Hidden trong terminal
}
```

**Ngày 28-30: Tổng Kết Tháng 1**
- Mini project: Full web app infrastructure (VPC + ECS + RDS + ALB)
- 3 environments: dev, staging, production
- Secrets in SSM, remote state per env
- Viết STAR story về bài học tháng 1

**Checklist Tháng 1:**
```
[ ] 3 environments hoạt động với isolated state
[ ] Secrets không bao giờ trong git
[ ] Terraform module library có VPC, compute, database
[ ] Có thể giải thích state management đầy đủ
[ ] Đã viết ít nhất 1 STAR story
```

---

## 📅 Tháng 2: Kỹ Năng Vận Hành (Ngày 31–60)

### Tuần 5 (Ngày 31–37): CI/CD Integration — Tích Hợp CI/CD

**Mục tiêu tuần:**
- Setup GitHub Actions pipeline cho Terraform
- Implement proper plan → review → apply workflow
- OIDC authentication (không dùng access keys)

**Lab chính:**
```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

permissions:
  id-token: write   # OIDC
  contents: read
  pull-requests: write

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/TerraformPlan
          aws-region: us-east-1

      - name: Terraform Setup
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Format
        run: terraform fmt -check -recursive infrastructure/

      - name: Terraform Init & Plan
        working-directory: infrastructure/environments/staging
        run: |
          terraform init
          terraform plan -out=plan.tfplan -no-color 2>&1 | tee plan.txt

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infrastructure/environments/staging/plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
            });

  terraform-apply:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: staging  # Requires manual approval via GitHub Environments
    steps:
      # ... similar setup ...
      - name: Terraform Apply
        working-directory: infrastructure/environments/staging
        run: |
          terraform init
          terraform apply -auto-approve
```

**Checklist Tuần 5:**
```
[ ] GitHub Actions pipeline chạy plan trên mọi PR
[ ] Plan output được post lên PR comment tự động
[ ] Apply chỉ chạy khi merge vào main
[ ] OIDC authentication (không có access key trong secrets)
[ ] Pipeline fail khi fmt check fail
```

---

### Tuần 6 (Ngày 38–44): Testing — Kiểm Thử

**Mục tiêu tuần:**
- Setup tfsec và Checkov trong CI/CD
- Viết test với terraform test (Terraform 1.6+)
- Hiểu test pyramid cho infrastructure

**Lab: Security scanning trong CI/CD**
```yaml
- name: tfsec Security Scan
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    working_directory: infrastructure/

- name: Checkov Scan
  uses: bridgecrewio/checkov-action@master
  with:
    directory: infrastructure/
    framework: terraform
    output_format: sarif
    output_file_path: results.sarif

- name: Upload SARIF — Static Analysis Results — to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
```

**Lab: Terraform native testing (1.6+)**
```hcl
# tests/vpc.tftest.hcl
run "vpc_exists" {
  command = plan

  variables {
    vpc_cidr    = "10.0.0.0/16"
    environment = "test"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block must be 10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.tags["ManagedBy"] == "terraform"
    error_message = "VPC must have ManagedBy=terraform tag"
  }
}
```

**Checklist Tuần 6:**
```
[ ] tfsec scan trong CI/CD, fail build khi có HIGH issues
[ ] Checkov compliance check chạy tự động
[ ] Viết ít nhất 3 terraform test assertions
[ ] Hiểu test pyramid: static → unit → integration
[ ] Security findings được upload lên GitHub Security tab
```

---

### Tuần 7 (Ngày 45–51): Monitoring & Drift Detection — Giám Sát & Phát Hiện Lệch

**Lab chính: Drift detection pipeline**
```bash
# Scheduled pipeline — chạy hàng ngày
# .github/workflows/drift-detection.yml
on:
  schedule:
    - cron: '0 8 * * *'   # 8am UTC hàng ngày

jobs:
  detect-drift:
    steps:
      - name: Terraform Plan (detect drift)
        run: |
          terraform plan -detailed-exitcode -no-color 2>&1 | tee plan.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT

      - name: Alert if drift detected
        if: steps.plan.outputs.exit_code == '2'
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "⚠️ Terraform drift detected in production!",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "Production infrastructure has drifted from Terraform code.\nPR: ${{ github.server_url }}/${{ github.repository }}/actions"
                }
              }]
            }
```

**Checklist Tuần 7:**
```
[ ] Drift detection chạy scheduled hàng ngày
[ ] Slack/email alert khi phát hiện drift
[ ] Hiểu 3 options xử lý drift (sync state, fix code, exclude)
[ ] Infracost setup và thấy cost estimate trên PR
[ ] Biết resource tagging strategy
```

---

### Tuần 8 (Ngày 52–60): Advanced Patterns — Mẫu Nâng Cao

**Ngày 52-54: Dynamic Blocks & For Expressions**
- Lab: Viết Security Group với dynamic ingress rules
- Lab: For expressions để transform data

**Ngày 55-57: Multi-region Setup**
```hcl
# Lab: Primary + DR region
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "ap-southeast-1"
}

module "primary" {
  source    = "./modules/app"
  providers = { aws = aws.primary }
  environment = "production"
}

module "dr" {
  source    = "./modules/app"
  providers = { aws = aws.dr }
  environment = "dr"
}
```

**Ngày 58-60: Tổng Kết Tháng 2**
- Review toàn bộ infrastructure code
- Chạy security scan và fix tất cả HIGH findings
- Viết thêm 2 STAR stories

**Checklist Tháng 2:**
```
[ ] CI/CD pipeline hoàn chỉnh với plan → approve → apply
[ ] Security scanning automatic trong mọi PR
[ ] Drift detection chạy hàng ngày
[ ] Infrastructure đã có tests với assertions
[ ] Multi-region deployment thử nghiệm
[ ] Có ít nhất 3 STAR stories
```

---

## 📅 Tháng 3: Chuyên Sâu & Chuẩn Bị Phỏng Vấn (Ngày 61–90)

### Tuần 9 (Ngày 61–67): Troubleshooting — Xử Lý Sự Cố

**Ngày 61-62: Simulated Incidents — Mô Phỏng Sự Cố**
```bash
# Lab 1: Force state corruption và practice recovery
# Lab 2: Create dependency cycle và debug nó
# Lab 3: Simulate provider timeout và implement retry logic
# Lab 4: Import orphaned resources
```

**Ngày 63-64: Advanced Debugging**
```bash
# Practice với TF_LOG=DEBUG
TF_LOG=DEBUG terraform apply 2>&1 | grep -E "Error|Warning|aws.PutItem"

# Đọc và hiểu provider crash logs
# Biết cách report issue lên Terraform/provider GitHub
```

**Ngày 65-67: Production Readiness Checklist**
- Đọc: `09-troubleshooting/6-production-checklist.md`
- Apply checklist cho toàn bộ infrastructure lab của bạn
- Document mọi item không pass và cách fix

**Checklist Tuần 9:**
```
[ ] Thực hành state recovery ít nhất 3 lần
[ ] Debug dependency cycle thực tế
[ ] Import ít nhất 5 resources từ "manual" creates
[ ] Infrastructure lab pass 100% production checklist
[ ] Biết sử dụng TF_LOG để debug provider issues
```

---

### Tuần 10 (Ngày 68–74): Terraform Ecosystem — Hệ Sinh Thái

**Ngày 68-70: Terragrunt Deep Dive**
```hcl
# Lab: Setup Terragrunt cho multi-env
# root terragrunt.hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "company-terraform-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "us-east-1"
  }
}

# environments/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}
```

**Ngày 71-72: OpenTofu — Alternative Open-Source**
```bash
# Lab: So sánh OpenTofu vs Terraform
# Install OpenTofu
brew install opentofu

# Chạy code Terraform với OpenTofu (thường compatible)
tofu init
tofu plan
tofu apply
```

**Ngày 73-74: CDK for Terraform — Viết Terraform bằng Python**
```python
# Lab: Viết infrastructure bằng Python
from constructs import Construct
from cdktf import App, TerraformStack, TerraformOutput
from cdktf_cdktf_provider_aws.instance import Instance
from cdktf_cdktf_provider_aws.provider import AwsProvider

class MyStack(TerraformStack):
    def __init__(self, scope: Construct, id: str):
        super().__init__(scope, id)
        AwsProvider(self, "AWS", region="us-east-1")

        instance = Instance(self, "WebServer",
            ami="ami-0c55b159cbfafe1f0",
            instance_type="t3.micro"
        )

        TerraformOutput(self, "instance_ip",
            value=instance.public_ip
        )
```

---

### Tuần 11 (Ngày 75–81): Interview Preparation — Chuẩn Bị Phỏng Vấn

**Ngày 75-76: Master the Top 20**
- Đọc lại: `1-INTERVIEW_GUIDE.md` với tinh thần "có thể giải thích cho 5-year-old"
- Luyện tập: Nói to đáp án cho từng câu
- Record video: Tự quay lại để nghe lại và cải thiện

**Ngày 77-78: STAR Stories**
- Hoàn thiện 3 STAR stories từ trải nghiệm thực tế
- Mỗi story phải có:
  - Số cụ thể (downtime, %, tiền tiết kiệm)
  - Technical depth (commands, tools, decisions)
  - Lesson learned (growth mindset)
- Tập kể to, đo thời gian (target: 3-4 phút mỗi story)

**Ngày 79-80: System Design Practice**
- Luyện tập 2 bài toán từ `3-system-design-scenarios.md`
- Vẽ diagram trong 5 phút
- Giải thích trade-offs rõ ràng
- Mock interview với đồng nghiệp

**Ngày 81: Mock Interview Day 1**
```
Agenda mock interview (90 phút):
- 15 phút: Introduction, experience overview
- 30 phút: Technical Q&A từ top 20 questions
- 20 phút: System design bài toán mới (không chuẩn bị trước)
- 20 phút: STAR behavioral questions
- 5 phút: Questions from interviewee
```

**Checklist Tuần 11:**
```
[ ] Có thể trả lời top 20 câu hỏi trong < 3 phút mỗi câu
[ ] 3 STAR stories với số liệu cụ thể
[ ] Vẽ được system design diagram trong < 10 phút
[ ] Xong ít nhất 1 mock interview đầy đủ
[ ] Biết giải thích trade-offs của mọi lựa chọn kỹ thuật
```

---

### Tuần 12 (Ngày 82–90): Final Sprint — Nước Rút Cuối

**Ngày 82-84: Portfolio Project — Dự Án Portfolio**

Xây dựng end-to-end project có thể show trong phỏng vấn:

```
Project: "Production-grade web application infrastructure"
Components:
├── Networking: VPC, subnets, NAT gateway, security groups
├── Compute: ECS Fargate với ALB và auto-scaling
├── Database: RDS PostgreSQL với Multi-AZ — Multi Availability Zone
├── Cache: ElastiCache Redis
├── Storage: S3 với CloudFront CDN
├── Security: IAM roles, KMS encryption, WAF — Web Application Firewall
├── Monitoring: CloudWatch dashboards, SNS alerts
└── CI/CD: GitHub Actions với plan/apply workflow

Infrastructure code:
├── modules/ (reusable components)
├── environments/dev/ (khởi đầu nhỏ)
├── environments/production/ (full setup)
└── README.md (hướng dẫn setup)
```

**Ngày 85-87: Documentation & Polish — Hoàn Thiện**
- Viết README chi tiết cho portfolio project
- Document: architecture decisions — Quyết định kiến trúc, trade-offs
- Create: architecture diagram (draw.io hoặc Mermaid)
- Prepare: "walk through" cách giải thích code trong phỏng vấn

**Ngày 88-89: Final Review — Ôn Tập Cuối**

```bash
# Run full checklist:
# 1. tfsec scan — phải 0 HIGH issues
tfsec .

# 2. Checkov scan
checkov -d . --framework terraform

# 3. terraform fmt check
terraform fmt -check -recursive

# 4. terraform validate
terraform validate

# 5. terraform plan — phải "No changes" cho existing envs
terraform plan

# 6. Review tất cả outputs có meaningful?
terraform output

# 7. Verify destroy works (staging only)
terraform destroy  # staging environment
```

**Ngày 90: Confidence Day — Ngày Tự Tin**

Hôm nay KHÔNG học thêm gì mới. Thay vào đó:
1. Đọc lại STAR stories của bạn — biết chúng như thuộc lòng
2. Ôn 10 câu hỏi bạn chưa tự tin nhất
3. Ngủ đủ giấc
4. Tin tưởng vào 90 ngày đã chuẩn bị

---

## 📊 Tracking Progress — Theo Dõi Tiến Độ

### Weekly Self-Assessment — Đánh Giá Bản Thân Hàng Tuần

```
Sau mỗi tuần, tự hỏi:
1. Tôi có thể giải thích topic tuần này cho ai không biết gì không?
2. Tôi đã thực hành hands-on bao nhiêu giờ?
3. Có concept nào tôi vẫn còn mờ không?
4. Tôi có thể code ví dụ từ đầu không (không copy-paste)?
```

### Milestone Checkpoints — Điểm Kiểm Tra Cột Mốc

```
✅ Ngày 7:  Terraform first resource chạy được
✅ Ngày 14: Remote backend với locking hoạt động
✅ Ngày 21: Module đầu tiên có README và tests
✅ Ngày 30: 3 environments với isolated state
✅ Ngày 44: CI/CD pipeline hoàn chỉnh
✅ Ngày 60: Infrastructure pass security scan
✅ Ngày 75: Top 20 câu hỏi trả lời được
✅ Ngày 81: Mock interview hoàn thành
✅ Ngày 90: Portfolio project deployed và documented
```

---

## ⏱️ Phân Bổ Thời Gian Thực Tế

```
Thời gian tối thiểu mỗi ngày: 1.5 giờ
Thời gian lý tưởng: 2-3 giờ

Phân bổ:
- Đọc lý thuyết: 30 phút
- Hands-on lab: 60-90 phút
- Review & notes: 15-30 phút

Cuối tuần:
- Deep dive project: 3-4 giờ/ngày
- Catch up nếu bị trễ trong tuần
```

---

## 💡 Lời Khuyên Thực Tế

```
1. Lab trước, lý thuyết sau: Tạo resource trước, đọc docs sau.
   Lỗi thực tế dạy tốt hơn docs.

2. Commit mọi thứ vào git: Code lab, notes, STAR stories.
   Git history là bằng chứng learning của bạn.

3. Break things intentionally — Cố tình phá để học:
   "Điều gì xảy ra nếu tôi xóa state file?"
   "Điều gì xảy ra nếu tôi apply hai lần cùng lúc?"

4. Đừng skip destroy: Sau mỗi lab, terraform destroy.
   AWS Free Tier có limits — vượt quá sẽ tốn tiền.

5. Kết bạn với error messages: Đọc kỹ, Google cụ thể,
   đọc provider GitHub issues khi cần.

6. Teach to learn — Dạy để học:
   Viết blog post hoặc giải thích cho đồng nghiệp sau mỗi
   tuần. Teaching solidifies understanding.
```

---

## 📚 Tài Liệu Tham Khảo Theo Giai Đoạn

### Tháng 1 — Nền Tảng
- HashiCorp Learn: learn.hashicorp.com/terraform
- Terraform docs: developer.hashicorp.com/terraform/docs
- Book: "Terraform: Up and Running" — Chương 1-4

### Tháng 2 — Vận Hành
- Gruntwork Blog: blog.gruntwork.io
- Anton Babenko's Terraform content (YouTube, blog)
- AWS Provider Docs: registry.terraform.io/providers/hashicorp/aws

### Tháng 3 — Nâng Cao
- Book: "Terraform: Up and Running" — Chương 5-10
- HashiCorp Vault docs (nếu dùng Vault)
- Terratest docs: github.com/gruntwork-io/terratest
- OpenTofu docs: opentofu.org/docs

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
