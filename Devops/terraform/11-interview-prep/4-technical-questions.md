# Q&A Kỹ Thuật Tổng Hợp

> Bộ câu hỏi & đáp án kỹ thuật chi tiết, phân theo chủ đề. Tập trung vào những câu hỏi đòi hỏi hiểu sâu, không chỉ thuộc lòng định nghĩa.

---

## 🗂️ Mục Lục

1. [HCL & Syntax](#1-hcl--syntax)
2. [State & Backend](#2-state--backend)
3. [Modules & Reusability](#3-modules--reusability)
4. [Providers & Resources](#4-providers--resources)
5. [Functions & Expressions](#5-functions--expressions)
6. [Workspaces & Environments](#6-workspaces--environments)
7. [CI/CD & Automation](#7-cicd--automation)
8. [Security & Compliance](#8-security--compliance)
9. [Performance & Scale](#9-performance--scale)
10. [Debugging & Troubleshooting](#10-debugging--troubleshooting)

---

## 1. HCL & Syntax

### Q: Sự khác biệt giữa `variable`, `local`, và `output`?

```hcl
# variable: Input từ bên ngoài (user, tfvars, environment)
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Must be t3.micro, t3.small, or t3.medium."
  }
}

# local: Computed values chỉ dùng trong module này
# Dùng khi cần transform hoặc combine variables
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }
  instance_name = "${var.project}-${var.environment}-web"
}

# output: Expose values ra ngoài module để module khác hoặc
# root module sử dụng
output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of web instance"
  sensitive   = false
}
```

**Nguyên tắc:**
- `variable`: cho external configuration
- `local`: cho DRY — Don't Repeat Yourself — và computed values
- `output`: cho cross-module communication và user information

---

### Q: `terraform.tfvars` vs `*.auto.tfvars` vs `TF_VAR_*` — thứ tự ưu tiên?

```
Thứ tự ưu tiên (từ thấp đến cao — cao hơn ghi đè thấp hơn):

1. Default values trong variable block
2. terraform.tfvars (auto-loaded)
3. terraform.tfvars.json (auto-loaded)
4. *.auto.tfvars files (auto-loaded, theo alphabet)
5. *.auto.tfvars.json files
6. -var-file="file.tfvars" (command line flag)
7. -var="name=value" (command line flag)
8. TF_VAR_name environment variables

Thực tế:
- terraform.tfvars: default values cho local development
- env.auto.tfvars: per-environment values
- TF_VAR_*: CI/CD secrets (không commit vào git)
- -var-file: Khi cần chỉ định explicit file
```

---

### Q: `templatefile()` vs `file()` — dùng khi nào?

```hcl
# file(): Đọc file nguyên xi, không template substitution
resource "aws_s3_object" "static" {
  key     = "config.json"
  content = file("${path.module}/config.json")
}

# templatefile(): Đọc file và substitute variables
# Dùng cho user_data, cloud-init scripts, config templates
resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/user_data.tpl", {
    db_host     = aws_db_instance.main.address
    app_version = var.app_version
    environment = var.environment
  })
}

# user_data.tpl:
# #!/bin/bash
# echo "DB_HOST=${db_host}" >> /etc/environment
# echo "ENV=${environment}" >> /etc/environment
# docker run -e DB_HOST=${db_host} myapp:${app_version}
```

---

## 2. State & Backend

### Q: Terraform xử lý sensitive values trong state file như thế nào?

```hcl
# Khai báo sensitive variable
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "main" {
  password = var.db_password
}

# THỰC TẾ QUAN TRỌNG:
# sensitive = true chỉ ẩn khỏi plan/apply OUTPUT
# Giá trị VẪN được lưu trong state file dưới dạng plaintext

# State file sẽ chứa:
# "password": "MySecureP@ssword123"
# (không encrypted trong JSON)

# Biện pháp bảo vệ:
# 1. Encrypt backend (S3 với KMS)
# 2. Restrict access to state file (S3 bucket policy, IAM)
# 3. Không dùng terraform output cho sensitive values
#    trừ khi thực sự cần

# Kiểm tra sensitive output:
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true  # Ẩn khỏi terminal nhưng vẫn trong state
}
```

---

### Q: `terraform state mv` dùng để làm gì? Khi nào cần?

```bash
# terraform state mv: Đổi tên resource trong state
# MÀ KHÔNG destroy và recreate resource thực tế

# Use cases:
# 1. Refactor resource name
terraform state mv aws_instance.web aws_instance.web_server

# 2. Di chuyển resource vào module
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# 3. Di chuyển resource ra khỏi module
terraform state mv module.old_module.aws_s3_bucket.data aws_s3_bucket.data

# Lưu ý quan trọng:
# - Phải update code để match new address
# - Plan sau khi mv phải thấy "No changes"
# - Terraform 1.1+ khuyến nghị dùng moved block thay thế:

# moved block (idempotent — bất biến, declarative — khai báo):
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
# Ưu điểm moved block:
# - Có thể commit vào git (ai cũng thấy lịch sử rename)
# - Idempotent (chạy nhiều lần không sao)
# - Hoạt động trong module khi upgrade version
```

---

### Q: Terraformstate pull/push làm gì? Khi nào dùng?

```bash
# terraform state pull: Download current state từ backend về stdout
terraform state pull > backup.tfstate

# terraform state push: Upload local state lên backend
# NGUY HIỂM — CHỈ DÙNG KHI THỰC SỰ BIẾT MÌNH ĐANG LÀM GÌ
terraform state push repaired.tfstate

# Use cases hợp lệ:
# 1. Backup trước khi thao tác nguy hiểm
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate

# 2. Migrate state giữa backends
# Old backend → pull → push → New backend

# 3. Manual state repair (sau corruption)
# - Pull corrupted state
# - Edit JSON manually
# - Push repaired state

# KHÔNG dùng push thường xuyên trong workflow bình thường
# Luôn backup trước khi push
```

---

## 3. Modules & Reusability

### Q: Terraform Registry vs Private Module Registry — khi nào dùng gì?

```
Public Terraform Registry (registry.terraform.io):
✅ Dùng khi:
  - Cần module phổ biến, đã được test bởi community
  - Quick start cho common patterns
  - Không có sensitive business logic
  Ví dụ: terraform-aws-modules/vpc/aws, terraform-aws-modules/eks/aws

❌ Không dùng khi:
  - Module chứa company-specific patterns
  - Business logic độc quyền
  - Compliance requirements prohibit external dependencies
  - Need full control over module updates

Private Module Registry (Terraform Cloud / Artifactory / GitLab):
✅ Dùng khi:
  - Company-specific patterns cần reuse
  - Cần control versioning chặt chẽ
  - Compliance: không dùng external dependencies
  - Large team cần discoverability — Khả năng khám phá

Alternative cho small teams — Nhóm nhỏ:
- Git registry trong source module:
module "vpc" {
  source  = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"
}
# Đơn giản, không cần registry infrastructure
# Phù hợp cho team < 20 người
```

---

### Q: Làm thế nào để test Terraform modules?

```go
// Terratest — Go-based testing framework — Framework test bằng Go
// File: test/vpc_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
  "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
  t.Parallel()

  terraformOptions := &terraform.Options{
    TerraformDir: "../modules/vpc",
    Vars: map[string]interface{}{
      "vpc_cidr":     "10.0.0.0/16",
      "environment":  "test",
      "region":       "us-east-1",
    },
  }

  // Cleanup sau khi test xong (kể cả khi test fail)
  defer terraform.Destroy(t, terraformOptions)

  // Deploy
  terraform.InitAndApply(t, terraformOptions)

  // Lấy output
  vpcId := terraform.Output(t, terraformOptions, "vpc_id")

  // Assertions
  assert.NotEmpty(t, vpcId)

  // Verify trực tiếp trên AWS
  vpc := aws.GetVpcById(t, vpcId, "us-east-1")
  assert.Equal(t, "10.0.0.0/16", aws.GetVpcCidrBlock(t, vpcId, "us-east-1"))
  assert.True(t, vpc.State == "available")
}
```

**Testing pyramid cho Terraform:**
```
Level 1 — Static (nhanh, free, không cần cloud):
  terraform validate + fmt
  tflint
  tfsec / Checkov
  OPA / Conftest policy check

Level 2 — Unit test (deploy isolated module):
  Terratest với isolated resources
  Thường tốn tiền cloud nhưng ít hơn integration

Level 3 — Integration test (full stack deploy):
  Deploy complete environment
  Run application tests
  Verify outputs và behaviors
  Tốn kém nhất — chạy ít thường xuyên hơn
```

---

## 4. Providers & Resources

### Q: Lifecycle meta-arguments — Các đối số meta về vòng đời — là gì?

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    # create_before_destroy: Tạo mới trước khi xóa cũ
    # Dùng cho zero-downtime replacement
    # (thay vì destroy rồi create)
    create_before_destroy = true

    # prevent_destroy: Ngăn chặn terraform destroy
    # Dùng cho stateful resources: databases, storage
    # Terraform sẽ error thay vì destroy
    prevent_destroy = true

    # ignore_changes: Ignore drift với listed attributes
    # Dùng khi attribute được quản lý ngoài Terraform
    # (ví dụ: auto-scaling thay đổi desired_count)
    ignore_changes = [
      tags["LastDeployedBy"],
      user_data,  # AMI baking thay đổi nhưng không cần redeploy
    ]

    # replace_triggered_by: Force replacement khi resource khác thay đổi
    # Terraform 1.2+
    replace_triggered_by = [
      aws_security_group.web.id  # Replace instance khi SG thay đổi
    ]
  }
}
```

---

### Q: Data sources — Nguồn dữ liệu — vs Resources — Tài nguyên — khác gì?

```hcl
# Resource: Terraform quản lý (tạo, sửa, xóa)
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-company-data"
}

# Data Source: Terraform chỉ đọc (lookup existing resources)
# Không tạo hay sửa gì

# Use case 1: Lookup AMI mới nhất
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use case 2: Lookup VPC không do Terraform tạo
data "aws_vpc" "existing" {
  tags = {
    Name = "production-vpc"
  }
}

# Use case 3: Cross-state reference
# Đọc outputs từ state file khác
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "company-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_id
}
```

---

## 5. Functions & Expressions

### Q: Các functions quan trọng nhất trong Terraform?

```hcl
locals {
  # String functions — Hàm chuỗi
  upper_env = upper(var.environment)           # "PRODUCTION"
  trimmed   = trimspace("  hello  ")           # "hello"
  replaced  = replace("hello world", "world", "terraform")  # "hello terraform"

  # Collection functions — Hàm tập hợp
  merged_tags = merge(var.common_tags, { Name = var.resource_name })
  unique_azs  = distinct(["us-east-1a", "us-east-1b", "us-east-1a"])
  flat_list   = flatten([["a", "b"], ["c", "d"]])  # ["a", "b", "c", "d"]

  # Numeric functions — Hàm số
  max_size = max(var.min_capacity, 10)
  clamped  = min(max(var.desired, 1), 100)   # giữ trong range [1, 100]

  # Type conversion — Chuyển đổi kiểu
  as_string = tostring(42)
  as_list   = tolist(toset(["a", "b", "a"]))  # deduplicate

  # Encoding — Mã hóa
  b64_encoded = base64encode("hello terraform")
  json_string = jsonencode({ key = "value", list = [1, 2, 3] })
  from_json   = jsondecode("{\"key\": \"value\"}")

  # Conditional — Điều kiện
  env_suffix = var.environment == "production" ? "" : "-${var.environment}"
  name       = "myapp${local.env_suffix}"
  # production: "myapp"
  # staging:    "myapp-staging"

  # Try/can — Xử lý lỗi gracefully — Xử lý lỗi một cách khéo léo
  safe_lookup = try(var.optional_map["key"], "default_value")
  is_valid    = can(regex("^[a-z]+$", var.name))
}
```

---

### Q: `for` expressions — Biểu thức for — dùng như thế nào?

```hcl
variable "users" {
  default = ["alice", "bob", "charlie"]
}

variable "services" {
  default = {
    web = { port = 80, protocol = "tcp" }
    api = { port = 8080, protocol = "tcp" }
    db  = { port = 5432, protocol = "tcp" }
  }
}

locals {
  # List comprehension — Biểu thức danh sách
  upper_users = [for u in var.users : upper(u)]
  # ["ALICE", "BOB", "CHARLIE"]

  # Map comprehension — Biểu thức bản đồ
  user_emails = { for u in var.users : u => "${u}@company.com" }
  # { alice = "alice@company.com", bob = "bob@company.com", ... }

  # Filtering — Lọc
  long_names = [for u in var.users : u if length(u) > 4]
  # ["alice", "charlie"]

  # Nested iteration — Lặp lồng nhau
  service_ports = [
    for name, config in var.services : {
      name     = name
      port     = config.port
      protocol = config.protocol
    }
  ]

  # Flatten nested structures — Làm phẳng cấu trúc lồng nhau
  # Dùng với for_each để tạo multiple resources từ nested data
  all_rules = flatten([
    for sg_name, sg in var.security_groups : [
      for rule in sg.rules : {
        sg_name  = sg_name
        port     = rule.port
        protocol = rule.protocol
      }
    ]
  ])
}
```

---

## 6. Workspaces & Environments

### Q: Terraform Workspaces — Không Gian Làm Việc — có những hạn chế gì?

```
Workspaces LÀ GÌ:
- Cơ chế của Terraform để có nhiều state trong cùng một backend config
- terraform workspace new staging → tạo state file mới riêng
- Mặc định: workspace "default"

Workspaces KHÔNG PHẢI LÀ:
- Full environment isolation — Cô lập hoàn toàn
- Solution cho different configs per environment
- Best practice cho production systems (theo HashiCorp)

Hạn chế thực tế:

1. Không enforce different permissions:
   Người có quyền apply "default" workspace cũng có quyền
   apply "production" workspace — SECURITY RISK

2. Code không thể khác nhau giữa workspaces (dễ):
   Phải dùng awkward conditionals:
   locals {
     instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
   }
   # Rất messy với nhiều resources và envs

3. State files trong cùng bucket:
   s3://bucket/env:/production/terraform.tfstate
   s3://bucket/env:/staging/terraform.tfstate
   # Không có separation ở infrastructure level

4. Dễ apply nhầm workspace:
   terraform workspace select production  # OK
   # Sau đó quên, apply staging config vào production

Khi nên dùng Workspaces:
✅ Testing thêm resources trong cùng environment
✅ Developer cần isolated copy để thử nghiệm
✅ CI/CD per-PR environments (ephemeral — tạm thời)

Khi KHÔNG nên dùng:
❌ dev/staging/production separation
❌ Multi-team environments
❌ Anywhere security isolation matters
```

---

## 7. CI/CD & Automation

### Q: Atlantis — Tự Động Hóa Terraform qua Pull Request — hoạt động như thế nào?

```
Atlantis là webhook server nhận events từ GitHub/GitLab/Bitbucket
và chạy Terraform commands khi comment lên PR.

Workflow:
Developer push code → PR opened
  ↓
GitHub sends webhook → Atlantis nhận event
  ↓
Atlantis tự động chạy: terraform plan
  ↓
Post plan output lên PR comment
  ↓
Reviewer comment: "atlantis apply" hoặc "atlantis apply -d envs/prod"
  ↓
Atlantis chạy: terraform apply
  ↓
Post apply results lên PR comment
  ↓
PR auto-merged sau apply thành công

Atlantis config (atlantis.yaml):
version: 3
projects:
  - name: production-vpc
    dir: environments/production/vpc
    workspace: default
    autoplan:
      when_modified: ["**/*.tf", "../../modules/**/*.tf"]
    apply_requirements:
      - approved           # Cần ít nhất 1 approval
      - mergeable          # PR không có conflicts
      - undiverged         # Branch up-to-date với main

Ưu điểm vs CI/CD thuần:
+ Audit trail: ai apply, khi nào, plan output là gì — ngay trên PR
+ Serialized applies: không thể 2 applies chạy đồng thời
+ Self-service cho developers (không cần DevOps run apply)
+ Lock mechanism: PR lock repository trong khi apply

Nhược điểm:
- Phải self-host Atlantis server (hoặc Runatlantis.io cloud)
- Cần cấu hình networking cho webhook (Atlantis phải nhận được từ GitHub)
- Không phù hợp nếu cần complex approval workflows
```

---

### Q: Làm sao tích hợp cost estimation — ước tính chi phí — vào CI/CD?

```bash
# Infracost: Tool phân tích Terraform plan và tính chi phí
# Chạy trong CI/CD, post kết quả lên PR comment

# GitHub Actions example:
- name: Setup Infracost
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Generate Infracost cost estimate
  run: |
    infracost breakdown \
      --path environments/production \
      --terraform-var-file production.tfvars \
      --format json \
      --out-file /tmp/infracost.json

- name: Post Infracost comment
  uses: infracost/actions/comment@v3
  with:
    path: /tmp/infracost.json
    behavior: update  # Update existing comment thay vì tạo mới

# Output ví dụ trên PR:
# Monthly cost estimate:
# + aws_instance.web: $73.00/month (t3.medium)
# + aws_db_instance.main: $145.00/month (db.t3.small)
# Total: +$218.00/month
# vs previous: $183.00/month (+$35.00, +19.1%)

# Policy có thể enforce:
# infracost comment --behavior update \
#   --policy-path policy/cost_policy.rego
# Fail PR nếu increase > $500/month
```

---

## 8. Security & Compliance

### Q: tfsec vs Checkov vs Terrascan — dùng cái nào?

```
tfsec:
- Focus: Security best practices cho Terraform
- Speed: Rất nhanh (Go binary)
- Coverage: AWS, Azure, GCP tốt
- Output: Terminal, JSON, SARIF (GitHub Security tab)
- Best for: Quick security gate trong CI/CD
tfsec .

Checkov:
- Focus: Security + compliance (CIS, PCI-DSS, HIPAA, NIST)
- Speed: Chậm hơn tfsec (Python)
- Coverage: Terraform + Kubernetes + CloudFormation + ARM
- Best for: Compliance reporting, multi-IaC environments
checkov -d . --framework terraform --check CKV_AWS_*

Terrascan:
- Focus: Policy as Code với OPA — Open Policy Agent
- Speed: Trung bình
- Coverage: Terraform, K8s, Docker, CloudFormation
- Best for: Custom policies, multi-cloud

Khi nào dùng gì:
- Quick security check: tfsec (nhanh nhất)
- Compliance reports: Checkov (framework coverage tốt nhất)
- Custom policies: OPA/Conftest với custom .rego rules
- Enterprise với existing OPA: Terrascan

Nhiều teams dùng KẾT HỢP:
tfsec (fast fail) + Checkov (compliance) trong CI/CD
```

---

## 9. Performance & Scale

### Q: Làm thế nào khi `terraform apply` rất chậm (hàng trăm resources)?

```bash
# Terraform refresh tất cả resources mỗi lần apply
# Với 500+ resources → slow API calls → chậm

# Giải pháp 1: -refresh=false (dùng state as-is)
terraform plan -refresh=false
# Nhanh hơn nhiều nhưng có thể miss real-world changes
# Chỉ dùng khi biết chắc không có drift

# Giải pháp 2: -target (chỉ apply specific resources)
terraform apply -target=module.new_feature
# Cẩn thận: dễ tạo ra partial state inconsistency
# Chỉ dùng khi có lý do rõ ràng

# Giải pháp 3: -parallelism (mặc định là 10)
terraform apply -parallelism=20
# Tăng parallelism = nhiều API calls đồng thời
# Cẩn thận: có thể trigger rate limiting

# Giải pháp 4: Split state (giải pháp lâu dài tốt nhất)
# Thay vì 1 state cho toàn bộ infrastructure:
# → State riêng cho networking (ít thay đổi)
# → State riêng cho per-service (thay đổi thường xuyên)

# Giải pháp 5: Lazy refresh với AWS provider
provider "aws" {
  # Một số operations không cần refresh
  skip_metadata_api_check = true
  skip_requesting_account_id = true  # Nhỏ nhưng tiết kiệm API call
}
```

---

### Q: Terraform provider versioning — Đánh số phiên bản provider — best practices?

```hcl
# versions.tf — luôn pin versions để reproducible builds
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
      # ~> 5.0 = >= 5.0.0, < 6.0.0 (minor version updates OK)
      # = "5.31.0" nếu cần exact pin
      # >= 5.0.0 nếu muốn flexible (không khuyến nghị)
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
}

# Lock file — terraform.lock.hcl (auto-generated, PHẢI COMMIT vào git)
# Lock file pin EXACT version và checksums — mã kiểm tra
# Đảm bảo mọi team member và CI/CD dùng cùng provider version
# Update lock file: terraform providers lock

# Tại sao phải commit lock file:
# - Reproducible builds — Build có thể tái tạo
# - Security: verify checksums của provider binaries
# - Consistency: không ai tự ý upgrade provider
```

---

## 10. Debugging & Troubleshooting

### Q: Các TF_LOG levels — Cấp độ log — và khi nào dùng?

```bash
# Log levels (từ ít chi tiết đến nhiều nhất):
# OFF → ERROR → WARN → INFO → DEBUG → TRACE

export TF_LOG=DEBUG
terraform apply 2> terraform_debug.log

# ERROR: Chỉ log errors nghiêm trọng
# WARN: Lỗi không nghiêm trọng, deprecation warnings
# INFO: High-level operation steps (mặc định khi TF_LOG=INFO)
# DEBUG: API requests/responses, provider plugin calls
# TRACE: Toàn bộ thông tin (rất verbose — rất chi tiết)

# Log riêng cho provider (không lẫn với Terraform core):
export TF_LOG_PROVIDER=DEBUG
# Xem AWS API calls cụ thể

# Ghi log ra file (không hiện trên terminal):
export TF_LOG_PATH="/tmp/terraform-$(date +%Y%m%d).log"

# Debug một apply cụ thể:
TF_LOG=DEBUG terraform apply 2>&1 | grep -E "RequestID|StatusCode|Error"

# Tắt log sau khi debug xong (QUAN TRỌNG — CI/CD không nên có TF_LOG):
unset TF_LOG
unset TF_LOG_PATH
```

---

### Q: Khi `terraform plan` báo resource sẽ bị replace — thay thế — mà bạn không muốn?

```bash
# Xác định lý do replace:
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan | jq '
  .resource_changes[] |
  select(.change.actions | contains(["delete", "create"])) |
  {
    address: .address,
    actions: .change.actions,
    before: .change.before,
    after: .change.after
  }'

# Thường thấy reason: "forces replacement"
# Nghĩa là attribute đó là ForceNew trong provider schema
# (thay đổi attribute này = phải tạo resource mới)

# Giải pháp tùy trường hợp:

# Trường hợp 1: Đổi tên resource (refactoring)
# KHÔNG sửa name trực tiếp
# DÙNG: moved block hoặc terraform state mv

# Trường hợp 2: Thực sự cần thay đổi ForceNew attribute
# Dùng create_before_destroy để minimize downtime:
lifecycle {
  create_before_destroy = true
}

# Trường hợp 3: Thay đổi là không cần thiết (false drift)
# Kiểm tra xem có phải provider bug không
# Dùng ignore_changes nếu attribute không quan trọng:
lifecycle {
  ignore_changes = [user_data]  # Nếu user_data change không cần redeploy
}

# Trường hợp 4: Không muốn replace production resource ngay bây giờ
# Tạm thời: terraform plan -target=<other resources> (skip resource này)
# Lâu dài: Schedule maintenance window cho replacement
```

---

## 📋 Quick Reference Card — Thẻ Tham Khảo Nhanh

```
COMMANDS QUAN TRỌNG:
terraform init              # Initialize — Khởi tạo working directory
terraform fmt -recursive    # Format tất cả files
terraform validate          # Validate syntax
terraform plan -out=p.out   # Plan và save
terraform show p.out        # Xem plan đã save
terraform apply p.out       # Apply từ saved plan
terraform apply -auto-approve  # Skip confirmation (CI/CD only)
terraform destroy -target=x    # Destroy specific resource
terraform state list        # Xem tất cả resources trong state
terraform state show <addr> # Xem chi tiết một resource
terraform state mv a b      # Rename trong state
terraform state rm <addr>   # Xóa khỏi state (không destroy)
terraform import <addr> <id># Import existing resource
terraform output            # Xem tất cả outputs
terraform output -json      # Output dưới dạng JSON
terraform providers lock    # Update lock file
terraform force-unlock <id> # Giải phóng stuck lock

DEBUGGING:
TF_LOG=DEBUG terraform plan 2>&1 | head -100
TF_LOG_PROVIDER=TRACE terraform apply 2> provider.log
terraform console           # Interactive REPL — Vòng lặp đọc-đánh giá-in

ENVIRONMENT VARIABLES QUAN TRỌNG:
TF_VAR_<name>=value         # Set variable
TF_WORKSPACE=<name>         # Set workspace
TF_CLI_ARGS_plan=-refresh=false  # Default args cho plan
TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache  # Cache providers
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
