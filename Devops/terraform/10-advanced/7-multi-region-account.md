# Kiến Trúc Đa Vùng & Đa Tài Khoản — Multi-region & Multi-account Architecture

> Quản lý hạ tầng Terraform ở quy mô enterprise — doanh nghiệp lớn — yêu cầu kiến trúc tốt cho **Multi-region** — Đa Vùng — và **Multi-account** — Đa Tài Khoản — để đạt được isolation — cô lập, blast radius reduction — giảm vùng ảnh hưởng sự cố, và compliance — tuân thủ quy định.

---

## 1. Tại Sao Cần Multi-account?

### Lợi Ích Của Cấu Trúc Multi-account

```
┌─────────────────────────────────────────────────────────┐
│                  AWS Organization                       │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │  Management │  │   Security  │  │    Log Archive  │ │
│  │   Account   │  │   Account   │  │    Account      │ │
│  │  (Root OU)  │  │             │  │                 │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │     Dev     │  │   Staging   │  │   Production    │ │
│  │   Account   │  │   Account   │  │    Account      │ │
│  │ 10.1.0.0/16 │  │ 10.2.0.0/16 │  │  10.0.0.0/16   │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐                       │
│  │   Sandbox   │  │  Shared     │                       │
│  │   Account   │  │  Services   │                       │
│  │ (Dev free)  │  │  Account    │                       │
│  └─────────────┘  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

| Lợi Ích | Giải Thích |
|---------|-----------|
| **Blast radius reduction** | Sự cố trong dev không ảnh hưởng production |
| **Security isolation** | IAM, network, resource isolation hoàn toàn |
| **Cost allocation** | Chi phí rõ ràng theo team/product |
| **Compliance** | PCI-DSS, HIPAA yêu cầu environment isolation |
| **Service limits** | Mỗi account có service limit riêng — không chia sẻ |

---

## 2. AWS Organizations Structure — Cấu Trúc Tổ Chức AWS

### Account Types — Các Loại Tài Khoản

```
Management Account (Root):
  - Quản lý tổ chức (AWS Organizations)
  - KHÔNG chứa workload thực tế
  - Chỉ SCP — Service Control Policies và billing

Security Account (Tooling):
  - IAM Identity Center — Single Sign-On cho toàn tổ chức
  - Security Hub, GuardDuty aggregation
  - Terraform state cho management plane

Log Archive Account:
  - CloudTrail logs từ tất cả account
  - Config rules aggregation
  - Security event logs — không ai được xóa logs ở đây

Shared Services Account:
  - Transit Gateway — Hub network kết nối các account
  - Shared VPN, Direct Connect
  - Internal DNS — Route53 Resolver
  - Container registry — ECR

Workload Accounts (Dev/Staging/Production):
  - Ứng dụng thực tế
  - Mỗi team có account riêng (hoặc chia sẻ theo product)
```

---

## 3. Terraform Multi-account Patterns — Kiểu Mẫu Đa Tài Khoản

### 3.1 Assume Role Pattern — Kiểu Mẫu Giả Định Vai Trò

```hcl
# providers.tf — Định nghĩa provider cho nhiều account

# Terraform chạy từ CI/CD với quyền của management account
# hoặc dùng IAM role với quyền AssumeRole vào các account con

provider "aws" {
  alias  = "management"
  region = "us-east-1"
  # Dùng credential hiện tại (không assume role)
}

provider "aws" {
  alias  = "security"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::${var.security_account_id}:role/TerraformRole"
    session_name = "TerraformSession-${terraform.workspace}"
    external_id  = var.external_id  # Thêm bảo mật — Confused Deputy protection
  }
}

provider "aws" {
  alias  = "production"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::${var.production_account_id}:role/TerraformRole"
    session_name = "TerraformSession-${terraform.workspace}"
  }
}

provider "aws" {
  alias  = "dev"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.dev_account_id}:role/TerraformRole"
    session_name = "TerraformSession"
  }
}
```

```hcl
# main.tf — Tạo resource trong account cụ thể

# Resource trong management account
resource "aws_organizations_account" "dev_account" {
  provider = aws.management

  name  = "dev-account"
  email = "aws-dev@example.com"
}

# Resource trong security account
resource "aws_securityhub_account" "security_hub" {
  provider = aws.security
}

# Resource trong production
resource "aws_vpc" "prod" {
  provider   = aws.production
  cidr_block = "10.0.0.0/16"

  tags = { Name = "prod-vpc", Account = "production" }
}

# Resource trong dev
resource "aws_vpc" "dev" {
  provider   = aws.dev
  cidr_block = "10.1.0.0/16"

  tags = { Name = "dev-vpc", Account = "dev" }
}
```

### 3.2 TerraformRole — IAM Role Trong Mỗi Account

```hcl
# Trong mỗi member account: tạo role để CI/CD assume
resource "aws_iam_role" "terraform" {
  name = "TerraformRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = [
            # Management account được phép assume role này
            "arn:aws:iam::${var.management_account_id}:root",
            # Hoặc specific role từ CI/CD
            "arn:aws:iam::${var.management_account_id}:role/CI-CD-Role",
          ]
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.external_id
          }
        }
      }
    ]
  })

  tags = { ManagedBy = "Terraform" }
}

# Grant permissions — Cấp quyền cho role
resource "aws_iam_role_policy_attachment" "terraform_admin" {
  role       = aws_iam_role.terraform.name
  # Trong production: dùng custom policy với least privilege
  # Trong dev/sandbox: có thể dùng AdministratorAccess
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
```

---

## 4. Terragrunt Multi-account Architecture — Kiến Trúc Đa Tài Khoản Với Terragrunt

Terragrunt là công cụ phổ biến nhất để quản lý multi-account với Terraform.

### Cấu Trúc Thư Mục

```
infrastructure/
├── terragrunt.hcl                    # Root config — cấu hình gốc
├── _envcommon/                       # Shared config — cấu hình chung
│   ├── vpc.hcl
│   ├── eks.hcl
│   └── rds.hcl
│
├── management/                       # Management account
│   ├── account.hcl
│   └── us-east-1/
│       ├── region.hcl
│       └── organizations/
│           └── terragrunt.hcl
│
├── production/                       # Production account
│   ├── account.hcl
│   └── us-east-1/
│       ├── region.hcl
│       ├── vpc/
│       │   └── terragrunt.hcl
│       ├── eks/
│       │   └── terragrunt.hcl
│       └── rds/
│           └── terragrunt.hcl
│
├── staging/                          # Staging account
│   ├── account.hcl
│   └── us-east-1/
│       ├── vpc/
│       │   └── terragrunt.hcl
│       └── eks/
│           └── terragrunt.hcl
│
└── dev/                              # Dev account
    ├── account.hcl
    └── us-east-1/
        ├── vpc/
        │   └── terragrunt.hcl
        └── eks/
            └── terragrunt.hcl
```

### `terragrunt.hcl` Root — Cấu Hình Gốc

```hcl
# infrastructure/terragrunt.hcl
locals {
  # Tự động đọc account config từ account.hcl gần nhất
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region

  # Đọc từ environment variable hoặc CI/CD
  management_account_id = get_env("MANAGEMENT_ACCOUNT_ID", "")
}

# Remote state riêng cho từng account + region + component
remote_state {
  backend = "s3"

  config = {
    encrypt        = true
    bucket         = "tfstate-${local.account_name}-${local.aws_region}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    dynamodb_table = "terraform-state-lock"

    role_arn = "arn:aws:iam::${local.account_id}:role/TerraformStateRole"
  }

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

# Provider config với assume role
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  assume_role {
    role_arn     = "arn:aws:iam::${local.account_id}:role/TerraformRole"
    session_name = "Terragrunt-${local.account_name}"
  }

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = "${local.account_name}"
      AccountId   = "${local.account_id}"
    }
  }
}
EOF
}
```

### `account.hcl` — Config Của Từng Account

```hcl
# infrastructure/production/account.hcl
locals {
  account_name = "production"
  account_id   = "111122223333"

  # Các setting riêng cho production
  instance_type   = "t3.large"
  min_nodes       = 3
  enable_deletion = false
}
```

```hcl
# infrastructure/dev/account.hcl
locals {
  account_name = "dev"
  account_id   = "444455556666"

  instance_type   = "t3.small"
  min_nodes       = 1
  enable_deletion = true  # Dev có thể destroy thoải mái
}
```

### Component-level Terragrunt Config

```hcl
# infrastructure/production/us-east-1/eks/terragrunt.hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  common_vars  = read_terragrunt_config(find_in_parent_folders("terragrunt.hcl"))
}

# Inherit từ root terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

# Module source với version pinning — ghim phiên bản
terraform {
  source = "git@github.com:example/terraform-modules.git//eks?ref=v3.2.1"
}

# Phụ thuộc vào VPC phải được tạo trước
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id     = "vpc-mock-id"
    subnet_ids = ["subnet-mock-1", "subnet-mock-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

# Input variables cho module
inputs = {
  cluster_name  = "eks-${local.account_vars.locals.account_name}"
  vpc_id        = dependency.vpc.outputs.vpc_id
  subnet_ids    = dependency.vpc.outputs.private_subnet_ids
  instance_type = local.account_vars.locals.instance_type
  min_nodes     = local.account_vars.locals.min_nodes
  max_nodes     = local.account_vars.locals.min_nodes * 3

  enable_cluster_autoscaler = true
  enable_aws_load_balancer_controller = true
}
```

---

## 5. Multi-region Architecture — Kiến Trúc Đa Vùng

### Active-Active vs Active-Passive

```
Active-Active — Tích cực-Tích cực:
  - Traffic được route đến cả hai region đồng thời
  - Phức tạp hơn: data replication, conflict resolution
  - Use case: Global applications, latency-critical services

Active-Passive — Tích cực-Thụ động:
  - Primary region xử lý tất cả traffic
  - Secondary region là warm standby (chờ sẵn)
  - Đơn giản hơn, chi phí thấp hơn
  - Use case: DR — Disaster Recovery, regulatory requirements
```

### Multi-region Provider Configuration

```hcl
# variables.tf
variable "primary_region" {
  default = "us-east-1"
}

variable "secondary_region" {
  default = "us-west-2"
}

variable "dr_region" {
  default = "eu-west-1"
}

# providers.tf
provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
}

provider "aws" {
  alias  = "dr"
  region = var.dr_region
}

# Route53 không có region → không cần alias cho global service
provider "aws" {
  alias  = "route53"
  region = "us-east-1"
}
```

### Multi-region RDS — Relational Database Service

```hcl
# Primary DB Cluster
resource "aws_rds_cluster" "primary" {
  provider               = aws.primary
  cluster_identifier     = "prod-db-primary"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  database_name          = "appdb"
  master_username        = var.db_username
  manage_master_user_password = true  # AWS Secrets Manager manages password

  # Enable global cluster để hỗ trợ cross-region replication
  global_cluster_identifier = aws_rds_global_cluster.main.id

  db_subnet_group_name   = aws_db_subnet_group.primary.name
  vpc_security_group_ids = [aws_security_group.rds_primary.id]

  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"

  lifecycle {
    prevent_destroy = true
  }
}

# Global Cluster — Quản lý cluster toàn cầu
resource "aws_rds_global_cluster" "main" {
  provider                  = aws.primary
  global_cluster_identifier = "prod-global-db"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "appdb"
  storage_encrypted         = true
}

# Secondary DB Cluster — Read replica ở region thứ hai
resource "aws_rds_cluster" "secondary" {
  provider               = aws.secondary
  cluster_identifier     = "prod-db-secondary"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"

  global_cluster_identifier = aws_rds_global_cluster.main.id

  db_subnet_group_name   = aws_db_subnet_group.secondary.name
  vpc_security_group_ids = [aws_security_group.rds_secondary.id]

  # Secondary cluster không có master credentials
  # Nhận data từ primary qua replication

  depends_on = [aws_rds_cluster_instance.primary]

  lifecycle {
    prevent_destroy = true
    ignore_changes = [
      replication_source_identifier,
    ]
  }
}
```

### Multi-region S3 Replication — Nhân Bản S3

```hcl
# Primary bucket
resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "my-app-data-primary"
}

resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  versioning_configuration { status = "Enabled" }
}

# Replica bucket
resource "aws_s3_bucket" "replica" {
  provider = aws.secondary
  bucket   = "my-app-data-replica"
}

resource "aws_s3_bucket_versioning" "replica" {
  provider = aws.secondary
  bucket   = aws_s3_bucket.replica.id
  versioning_configuration { status = "Enabled" }
}

# IAM Role for replication
resource "aws_iam_role" "replication" {
  provider = aws.primary
  name     = "s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
    }]
  })
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "primary_to_replica" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  role     = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD_IA"  # Infrequent Access — Truy cập không thường xuyên

      replication_time {
        status = "Enabled"
        time   { minutes = 15 }
      }

      metrics {
        status = "Enabled"
        event_threshold { minutes = 15 }
      }
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.primary,
    aws_s3_bucket_versioning.replica,
  ]
}
```

---

## 6. State Management Cho Multi-account/Multi-region

### State File Strategy — Chiến Lược File State

```
Option 1: Một state file cho mỗi account + region + component (Khuyến nghị)
  tfstate-production-us-east-1/vpc/terraform.tfstate
  tfstate-production-us-east-1/eks/terraform.tfstate
  tfstate-production-us-west-2/vpc/terraform.tfstate
  tfstate-dev-us-east-1/vpc/terraform.tfstate

  Ưu: Blast radius nhỏ, plan/apply nhanh, ít xung đột
  Nhược: Nhiều state file cần quản lý

Option 2: Một state file cho mỗi environment
  tfstate-production/terraform.tfstate
  tfstate-staging/terraform.tfstate
  tfstate-dev/terraform.tfstate

  Ưu: Đơn giản hơn
  Nhược: Blast radius lớn hơn, plan chậm với nhiều resource

Option 3: Monolithic (không khuyến nghị cho multi-account)
  tfstate/terraform.tfstate  ← Tất cả trong 1 file
  Chỉ phù hợp với project nhỏ
```

### Remote State Cross-account Access — Truy Cập State Giữa Các Tài Khoản

```hcl
# Trong production EKS config: cần biết VPC ID từ VPC state
data "terraform_remote_state" "vpc" {
  backend = "s3"

  config = {
    bucket   = "tfstate-production-us-east-1"
    key      = "vpc/terraform.tfstate"
    region   = "us-east-1"
    role_arn = "arn:aws:iam::${var.production_account_id}:role/TerraformStateReadRole"
  }
}

# Dùng output từ state khác
resource "aws_eks_cluster" "main" {
  name = "prod-eks"

  vpc_config {
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
    # ↑ Lấy từ VPC state thay vì hardcode
  }
}
```

---

## 7. Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ

SCP — Service Control Policies — là guardrails — hàng rào bảo vệ — áp dụng cho toàn bộ account, ngăn even root user làm một số hành động.

```hcl
# Tạo SCP bằng Terraform trong management account
resource "aws_organizations_policy" "deny_root_usage" {
  provider = aws.management

  name        = "DenyRootUserActions"
  description = "Ngăn sử dụng root user"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyRootUser"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = {
          StringLike = {
            "aws:PrincipalArn" = "arn:aws:iam::*:root"
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy" "deny_leave_org" {
  provider = aws.management
  name     = "DenyLeaveOrganization"
  type     = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Deny"
      Action   = "organizations:LeaveOrganization"
      Resource = "*"
    }]
  })
}

# Áp dụng SCP cho OU — Organizational Unit — hoặc account cụ thể
resource "aws_organizations_policy_attachment" "deny_root_production_ou" {
  provider  = aws.management
  policy_id = aws_organizations_policy.deny_root_usage.id
  target_id = aws_organizations_organizational_unit.production.id
}
```

---

## 8. AWS Control Tower Integration — Tích Hợp Control Tower

AWS Control Tower là dịch vụ thiết lập Landing Zone — Khu Đỗ — chuẩn cho multi-account.

```hcl
# Dùng Control Tower Account Factory để tạo account mới
# (Thường dùng qua Terraform module)
module "account_factory" {
  source = "git@github.com:aws-ia/terraform-aws-control_tower_account_factory.git//modules/aws-aft-account-request?ref=1.9.1"

  control_tower_parameters = {
    AccountEmail              = "workload-team-a@example.com"
    AccountName               = "workload-team-a"
    ManagedOrganizationalUnit = "Workloads/TeamA"
    SSOUserEmail              = "admin@example.com"
    SSOUserFirstName          = "Admin"
    SSOUserLastName           = "User"
  }

  account_tags = {
    Team        = "team-a"
    Environment = "production"
    CostCenter  = "CC-12345"
  }

  change_management_parameters = {
    change_requested_by = "HashiCorp Terraform"
    change_reason       = "Khởi tạo account cho Team A"
  }

  account_customizations_name = "workload-standard"
}
```

---

## 9. Checklist Multi-account Terraform

```
✅ Account Structure:
  □ Management account chỉ dùng cho AWS Organizations, SSO, billing
  □ Mỗi environment (dev/staging/prod) có account riêng
  □ Security và Log Archive account tách biệt
  □ SCP đã được thiết lập để ngăn xóa CloudTrail

✅ Terraform State:
  □ State bucket trong mỗi account (không cross-account)
  □ State locking với DynamoDB
  □ State bucket encryption và versioning
  □ State access qua IAM role, không dùng access key

✅ Authentication:
  □ CI/CD dùng IAM role, không dùng long-term access key
  □ TerraformRole trong mỗi account với least privilege
  □ External ID trong assume_role để chống Confused Deputy attack
  □ Session duration phù hợp (không quá dài)

✅ Network:
  □ VPC CIDR không overlap giữa các account
  □ Transit Gateway hoặc VPC Peering được plan trước
  □ DNS resolution cross-account được cấu hình

✅ Monitoring:
  □ CloudTrail bật ở tất cả region, tất cả account
  □ Log tập trung vào Log Archive account
  □ AWS Config Rules áp dụng qua organization
  □ GuardDuty enabled và aggregated vào Security account
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Thiết kế kiến trúc Terraform cho tổ chức có 50 AWS account?**

A: Dùng AWS Organizations với OU — Organizational Unit — theo function (Management, Security, Log Archive, Workloads). Terragrunt để quản lý DRY code với cấu trúc `account/region/component`. Mỗi account có TerraformRole riêng, CI/CD assume role vào từng account. State file tách biệt theo `account+region+component`. SCP bảo vệ guardrails cơ bản. Control Tower hoặc Landing Zone tự xây dựng để provision account mới tự động.

**Q: Tại sao không nên để tất cả resource của nhiều account trong một Terraform state?**

A: Blast radius — vùng ảnh hưởng — khi state bị corrupt hoặc plan/apply lỗi sẽ ảnh hưởng tất cả account. Plan time tăng theo số lượng resource. State locking block tất cả người dùng đồng thời. Khó kiểm soát access vì state chứa sensitive data của nhiều account.

**Q: External ID trong assume_role dùng để làm gì?**

A: External ID bảo vệ khỏi "Confused Deputy Problem" — vấn đề kẻ trung gian nhầm lẫn. Nếu không có External ID, một service (kẻ tấn công) có thể lừa AWS assume role bằng cách giả vờ là service hợp lệ. External ID là secret chỉ CI/CD và account biết, đảm bảo chỉ đúng caller mới assume được role.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [6-opentofu.md](6-opentofu.md) |
| → Phần tiếp | [11-interview-prep/README.md](../11-interview-prep/README.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐⭐ Chuyên Sâu — Cần Kinh Nghiệm Thực Tế
