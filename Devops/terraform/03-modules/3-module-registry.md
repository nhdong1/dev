# 3 — Terraform Registry — Kho Lưu Trữ Module

> Terraform Registry — Kho lưu trữ module Terraform — là nơi chia sẻ và tái sử dụng
> module công khai hoặc trong nội bộ tổ chức.
> Public Registry tại registry.terraform.io, Private Registry trong Terraform Cloud / HCP Terraform.

---

## 🌐 Public Registry — Kho Module Công Khai

### Truy Cập và Tìm Kiếm

```
URL: https://registry.terraform.io

Tìm module phổ biến:
  terraform-aws-modules/vpc/aws           ← VPC trên AWS
  terraform-aws-modules/eks/aws           ← EKS — Kubernetes trên AWS
  terraform-aws-modules/rds/aws           ← RDS — Database quan hệ
  terraform-google-modules/kubernetes-engine/google  ← GKE trên GCP
  Azure/compute/azurerm                   ← Compute trên Azure
```

### Cú Pháp Source Cho Registry Module

```hcl
# Định dạng: <namespace>/<module-name>/<provider>
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}
```

### Ví Dụ Dùng Các Module AWS Phổ Biến

```hcl
# EKS cluster — Kubernetes cluster trên AWS
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.29"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    general = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 5
      desired_size   = 2
    }
  }
}

# RDS PostgreSQL
module "db" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier        = "my-database"
  engine            = "postgres"
  engine_version    = "14"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  db_name  = "myapp"
  username = "admin"

  vpc_security_group_ids = [module.security_group.security_group_id]
  db_subnet_group_name   = module.vpc.database_subnet_group

  family               = "postgres14"
  major_engine_version = "14"
}
```

---

## 🔍 Cách Đọc Module Trên Registry

```
Trang module terraform-aws-modules/vpc/aws chứa:

  Readme tab       → Mô tả, usage examples
  Inputs tab       → Tất cả variables và defaults
  Outputs tab      → Tất cả outputs
  Resources tab    → Tất cả AWS resources sẽ được tạo
  Dependencies tab → Các modules khác được dùng
  Versions tab     → Lịch sử versions
```

---

## 🔐 Private Registry — Kho Module Nội Bộ

### Tại Sao Cần Private Registry

```
Lý do tổ chức cần private registry:
  1. Code nội bộ không muốn public — bảo mật kiến trúc
  2. Module tuân theo policy công ty (naming, tagging, security)
  3. Versioning và governance có kiểm soát
  4. Không phụ thuộc vào external registry (supply chain security)
  5. Cần authentication — Xác thực — để truy cập
```

### Terraform Cloud / HCP Terraform Private Registry

```hcl
# Cấu hình trong ~/.terraformrc
credentials "app.terraform.io" {
  token = "YOUR_TERRAFORM_CLOUD_TOKEN"
}
```

```hcl
# Gọi module từ private registry Terraform Cloud
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "~> 2.0"

  # Variables như bình thường
  vpc_cidr = "10.0.0.0/16"
}
```

### Git-Based Modules — Module Từ Git Repository

```hcl
# HTTPS từ GitHub — phổ biến nhất
module "vpc" {
  source = "github.com/my-org/terraform-modules//networking/vpc"
  # Dùng ?ref= để pin version
}

# SSH từ GitHub
module "vpc" {
  source = "git@github.com:my-org/terraform-modules.git//networking/vpc"
}

# Pin theo Git tag — cách dùng cho production
module "vpc" {
  source = "github.com/my-org/terraform-modules//networking/vpc?ref=v2.1.0"
}

# Pin theo commit hash — cách an toàn nhất
module "vpc" {
  source = "github.com/my-org/terraform-modules//networking/vpc?ref=a1b2c3d4"
}

# Pin theo branch — dùng để develop, KHÔNG dùng cho production
module "vpc" {
  source = "github.com/my-org/terraform-modules//networking/vpc?ref=feature/new-az-support"
}
```

> **Chú ý `//` (double slash):** Dùng `//` để chỉ subdirectory trong git repo.
> `github.com/org/repo//path/to/module` → lấy code trong thư mục `path/to/module`

---

## 📦 Publish Module Lên Public Registry

### Yêu Cầu Để Publish

```
1. GitHub repository với naming convention:
   terraform-<provider>-<name>
   Ví dụ: terraform-aws-vpc, terraform-google-gke

2. Repository phải public

3. Cấu trúc chuẩn:
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   ├── README.md (bắt buộc)
   └── examples/
       └── basic/

4. Git tags theo semantic versioning:
   v1.0.0, v1.1.0, v2.0.0

5. Đăng nhập registry.terraform.io bằng GitHub account
```

### Quy Trình Publish

```bash
# 1. Tạo repo đúng naming convention
git init terraform-aws-my-module
cd terraform-aws-my-module

# 2. Viết module code, tests, examples
# ... (tạo main.tf, variables.tf, outputs.tf, README.md)

# 3. Commit và push
git add .
git commit -m "feat: initial module release v1.0.0"
git push origin main

# 4. Tạo Git tag
git tag v1.0.0
git push origin v1.0.0

# 5. Trên registry.terraform.io:
#    - Đăng nhập bằng GitHub
#    - "Publish" → chọn repository
#    - Registry tự động detect tag v1.0.0
```

---

## 🏢 Xây Dựng Private Registry Với GitLab

```
GitLab hỗ trợ private Terraform module registry tích hợp sẵn.

Cấu trúc URL:
  gitlab.example.com/api/v4/projects/<PROJECT_ID>/packages/terraform/modules/<MODULE_NAME>/<PROVIDER>

Publish module lên GitLab Registry:
```

```bash
# Publish module qua GitLab CI/CD
terraform-module:
  stage: publish
  image: curlimages/curl:latest
  script:
    - |
      curl --header "Job-Token: ${CI_JOB_TOKEN}" \
           --upload-file my-module.tgz \
           "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/terraform/modules/my-module/aws/${MODULE_VERSION}/file"
```

```hcl
# Dùng module từ GitLab registry
terraform {
  required_providers {
    # Cấu hình provider cho GitLab
  }
}

module "vpc" {
  source  = "gitlab.example.com/my-group/vpc/aws"
  version = "1.0.0"
}
```

---

## 🔑 Authentication — Xác Thực Module Registry

### Terraform Cloud Token

```hcl
# ~/.terraformrc
credentials "app.terraform.io" {
  token = "YOUR_API_TOKEN"
}
```

### GitHub Token Cho Git-Based Modules

```bash
# Dùng GitHub token qua netrc
echo "machine github.com login TOKEN password x-oauth-basic" >> ~/.netrc

# Hoặc SSH key (khuyến nghị cho CI/CD)
# Cấu hình SSH key trong deploy key của repository
```

```hcl
# Trong CI/CD, dùng git config thay vì SSH
# (nếu dùng GitHub Actions)
git config --global url."https://token:${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
```

---

## 📊 So Sánh Các Nguồn Module

| Nguồn                     | Cú Pháp                              | Version Pin | Auth  | Dùng Khi                          |
| ------------------------- | ------------------------------------ | ----------- | ----- | --------------------------------- |
| Local path                | `./modules/vpc`                      | Không        | Không | Module trong cùng repo            |
| Public Registry           | `terraform-aws-modules/vpc/aws`      | `version =` | Không | Module open-source phổ biến       |
| Private Registry (TF Cloud)| `app.terraform.io/org/vpc/aws`      | `version =` | Token | Module nội bộ tổ chức lớn         |
| GitHub (tag)              | `github.com/org/repo//path?ref=v1.0` | `?ref=`     | Token | Module private, không dùng TF Cloud|
| GitHub (hash)             | `github.com/org/repo//path?ref=abc123`| `?ref=`    | Token | Cần immutable reference tuyệt đối |
| GitLab Registry           | `gitlab.com/group/vpc/aws`           | `version =` | Token | Dùng GitLab làm platform          |
| S3 Bucket                 | `s3::https://...`                    | URL path    | IAM   | Air-gapped — môi trường cô lập    |

---

## ⚠️ Security Considerations — Cân Nhắc Bảo Mật

### Supply Chain Attacks — Tấn Công Chuỗi Cung Ứng

```
Rủi ro khi dùng public modules:
  1. Module bị compromised — bị xâm phạm
  2. Module namespace bị squatting — chiếm đoạt namespace
  3. Module dependencies không được kiểm soát
  4. Breaking changes khi không pin version
```

### Giảm Thiểu Rủi Ro

```hcl
# 1. Pin version cụ thể cho production
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"    # Pin exact version, không dùng ~> hoặc >=
}

# 2. Dùng .terraform.lock.hcl — Dependency lock file
# Sau khi terraform init, commit file này vào git:
# .terraform.lock.hcl lưu hash của provider/module đã tải

# 3. Xem code trước khi dùng
# git clone module repo, review code, sau đó mirror vào private registry

# 4. Dùng private registry với audit trail — Nhật ký kiểm tra
# Mọi module access đều được log
```

### Lock File — File Khóa Phụ Thuộc

```
.terraform.lock.hcl được tạo sau terraform init.
File này chứa hash của providers (KHÔNG phải modules từ git).

Thực hành tốt:
  ✅ Commit .terraform.lock.hcl vào git
  ✅ Chạy terraform providers lock khi thêm provider mới
  ❌ Không bỏ .terraform.lock.hcl vào .gitignore
```

---

## 🔄 Workflow Thực Tế: Dùng Và Maintain Registry Module

### Thêm Module Mới Vào Project

```bash
# 1. Tìm module phù hợp trên registry.terraform.io
# 2. Đọc README, xem inputs/outputs/resources
# 3. Thêm vào main.tf với version pin

# 4. Chạy terraform init — tải module về
terraform init

# 5. Commit .terraform.lock.hcl (cho providers)
git add .terraform.lock.hcl
git commit -m "chore: add vpc module dependency"

# 6. Chạy terraform plan để review trước
terraform plan
```

### Nâng Cấp Module Version

```bash
# Xem version hiện tại
cat .terraform.lock.hcl | grep -A 5 "terraform-aws-modules"

# Cập nhật version trong source block
# version = "~> 5.0" → version = "~> 5.2"

# Tải version mới
terraform init -upgrade

# Review thay đổi cẩn thận
terraform plan

# Kiểm tra CHANGELOG của module trước khi upgrade major version
# https://github.com/terraform-aws-modules/terraform-aws-vpc/blob/master/CHANGELOG.md
```

---

## ✅ Checklist Registry Module

```
Khi dùng public module:
  [ ] Pin version (tránh unexpected upgrades — nâng cấp ngoài ý muốn)
  [ ] Đọc CHANGELOG khi upgrade major version
  [ ] Review resources sẽ được tạo (Resources tab trên registry)
  [ ] Commit .terraform.lock.hcl

Khi xây dựng private registry:
  [ ] Đặt tên repository đúng convention
  [ ] Module có README với usage example
  [ ] Module có tests
  [ ] Git tags theo semantic versioning
  [ ] Authentication được cấu hình đúng trong CI/CD

Security:
  [ ] Không dùng untagged git source cho production
  [ ] Review module code trước khi dùng lần đầu
  [ ] Có process để phát hiện module updates bảo mật
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Tiếp Theo:** [4-module-versioning.md](4-module-versioning.md) — Semantic Versioning và version strategy
