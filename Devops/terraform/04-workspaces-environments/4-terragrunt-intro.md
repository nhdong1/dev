# Terragrunt — Giải Pháp DRY Cho Đa Môi Trường

> Terragrunt — Wrapper Tool cho Terraform — giải quyết vấn đề lặp code khi quản lý nhiều môi trường, nhiều region, nhiều AWS account. Đây là công cụ tiêu chuẩn trong môi trường enterprise.

---

## 📚 Mục Lục

1. [Terragrunt là gì?](#terragrunt-là-gì)
2. [Vấn đề Terragrunt giải quyết](#vấn-đề-terragrunt-giải-quyết)
3. [Cài đặt và cấu hình cơ bản](#cài-đặt-và-cấu-hình-cơ-bản)
4. [terragrunt.hcl — File cấu hình chính](#terrагrunthcl-file-cấu-hình-chính)
5. [Cấu trúc thư mục với Terragrunt](#cấu-trúc-thư-mục-với-terragrunt)
6. [DRY — Don't Repeat Yourself — với include](#dry-dont-repeat-yourself-với-include)
7. [Dependency — Phụ thuộc — giữa modules](#dependency-phụ-thuộc-giữa-modules)
8. [Chạy nhiều modules cùng lúc](#chạy-nhiều-modules-cùng-lúc)
9. [Khi nào nên dùng Terragrunt](#khi-nào-nên-dùng-terragrunt)
10. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Terragrunt là gì?

**Terragrunt** là một thin wrapper — lớp bọc mỏng — bên ngoài Terraform, được phát triển bởi Gruntwork. Nó không thay thế Terraform mà bổ sung thêm tính năng:

- **DRY** — Don't Repeat Yourself — Không Lặp Lại — backend configuration
- **Dependency management** — Quản lý phụ thuộc — giữa các Terraform modules
- **Run multiple modules** — Chạy nhiều modules — cùng lúc với thứ tự đúng
- **Remote configuration** — Cấu hình từ xa — `include` và `read_terragrunt_config`
- **Hooks** — Móc nối — before/after apply (pre-apply, post-apply)

### Terragrunt vs Terraform

```
Terraform:
  Tốt ở: Quản lý một stack tài nguyên
  Yếu ở: Quản lý nhiều stack qua nhiều môi trường (lặp code nhiều)

Terragrunt:
  Tốt ở: Orchestrate — Điều phối — nhiều Terraform stacks
  Không thay thế: Vẫn cần Terraform để thực sự tạo tài nguyên
```

---

## Vấn đề Terragrunt giải quyết

### Vấn đề 1: Backend configuration lặp đi lặp lại

**Không có Terragrunt** — Mỗi môi trường phải copy toàn bộ backend config:

```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# environments/staging/backend.tf — Copy gần như y chang!
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "staging/terraform.tfstate"   # Chỉ khác chỗ này
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# environments/prod/backend.tf — Lại copy thêm lần nữa!
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"      # Và chỗ này
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**Với Terragrunt** — Định nghĩa một lần, dùng ở mọi nơi:

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### Vấn đề 2: Dependency giữa modules

```
Module vpc phải tạo xong trước khi module eks có thể dùng vpc_id.
Terraform không biết thứ tự này nếu các module ở thư mục khác nhau.
Terragrunt giải quyết: dependency block khai báo thứ tự rõ ràng.
```

### Vấn đề 3: Apply nhiều module theo thứ tự

```bash
# Không có Terragrunt: Phải làm thủ công từng bước
cd vpc && terraform apply
cd ../eks && terraform apply
cd ../rds && terraform apply
cd ../monitoring && terraform apply

# Với Terragrunt: Một lệnh, tự động đúng thứ tự
terragrunt run-all apply
```

---

## Cài đặt và cấu hình cơ bản

```bash
# macOS
brew install terragrunt

# Linux
# Tải binary từ GitHub releases
wget https://github.com/gruntwork-io/terragrunt/releases/download/v0.55.0/terragrunt_linux_amd64
chmod +x terragrunt_linux_amd64
mv terragrunt_linux_amd64 /usr/local/bin/terragrunt

# Kiểm tra phiên bản
terragrunt --version
```

### Các lệnh cơ bản

```bash
# Terragrunt wrap lại các lệnh Terraform
terragrunt init
terragrunt plan
terragrunt apply
terragrunt destroy

# Chạy cho tất cả modules trong thư mục (và sub-directories)
terragrunt run-all init
terragrunt run-all plan
terragrunt run-all apply

# Chỉ một module cụ thể
cd environments/prod/vpc
terragrunt apply
```

---

## terragrunt.hcl — File cấu hình chính

`terragrunt.hcl` là file cấu hình chính của Terragrunt, viết bằng HCL — HashiCorp Configuration Language.

### Cấu trúc cơ bản

```hcl
# terragrunt.hcl (đặt trong thư mục module)

# Chỉ định Terraform module nguồn
terraform {
  source = "../../modules//vpc"   # Dấu // phân cách path và subfolder
}

# Include config từ file cha (root terragrunt.hcl)
include "root" {
  path = find_in_parent_folders()
}

# Input variables — Biến đầu vào
inputs = {
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}
```

### Root terragrunt.hcl — Cấu hình gốc dùng chung

```hcl
# terragrunt.hcl (ở thư mục gốc)

locals {
  # Đọc cấu hình từ các file yaml theo hierarchy — Cấp bậc
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
  environment  = local.env_vars.locals.environment
}

# Remote state config — Cấu hình lưu state từ xa — dùng chung cho tất cả modules
remote_state {
  backend = "s3"
  config = {
    encrypt        = true
    bucket         = "mycompany-${local.account_id}-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    dynamodb_table = "terraform-state-lock"
  }
  generate = {
    path      = "backend.tf"    # Tự động tạo file backend.tf
    if_exists = "overwrite_terragrunt"
  }
}

# Provider config dùng chung
generate "provider" {
  path      = "provider.tf"    # Tự động tạo file provider.tf
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  default_tags {
    tags = {
      Environment = "${local.environment}"
      ManagedBy   = "terragrunt"
      AccountId   = "${local.account_id}"
    }
  }
}
EOF
}

# Inputs dùng chung cho tất cả modules
inputs = merge(
  local.account_vars.locals,
  local.region_vars.locals,
  local.env_vars.locals,
)
```

---

## Cấu trúc thư mục với Terragrunt

### Cấu trúc khuyến nghị

```
infra/
├── terragrunt.hcl              ← Root config — Cấu hình gốc dùng chung
│
├── modules/                    ← Terraform modules (không có terragrunt.hcl)
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   └── rds/
│
└── live/                       ← Môi trường thực tế (có terragrunt.hcl)
    ├── account.hcl              ← Account config
    │
    ├── us-east-1/               ← Region config
    │   ├── region.hcl
    │   │
    │   ├── dev/                 ← Environment
    │   │   ├── env.hcl
    │   │   ├── vpc/
    │   │   │   └── terragrunt.hcl
    │   │   ├── eks/
    │   │   │   └── terragrunt.hcl
    │   │   └── rds/
    │   │       └── terragrunt.hcl
    │   │
    │   ├── staging/
    │   │   ├── env.hcl
    │   │   ├── vpc/
    │   │   │   └── terragrunt.hcl
    │   │   └── ...
    │   │
    │   └── prod/
    │       ├── env.hcl
    │       ├── vpc/
    │       │   └── terragrunt.hcl
    │       └── ...
    │
    └── eu-west-1/               ← Region khác
        ├── region.hcl
        └── prod/
            └── ...
```

### Các file config theo hierarchy — Cấp bậc

**`live/account.hcl`:**
```hcl
locals {
  account_id   = "123456789012"
  account_name = "mycompany"
}
```

**`live/us-east-1/region.hcl`:**
```hcl
locals {
  aws_region = "us-east-1"
}
```

**`live/us-east-1/prod/env.hcl`:**
```hcl
locals {
  environment = "prod"
}
```

**`live/us-east-1/prod/vpc/terragrunt.hcl`:**
```hcl
include "root" {
  path = find_in_parent_folders()   # Tìm root terragrunt.hcl
}

terraform {
  source = "../../../../modules//vpc"
}

inputs = {
  vpc_cidr        = "10.2.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
  public_subnets  = ["10.2.101.0/24", "10.2.102.0/24", "10.2.103.0/24"]
}
```

---

## DRY — Don't Repeat Yourself — với include

### include block — Kế Thừa Cấu Hình

```hcl
# live/us-east-1/prod/eks/terragrunt.hcl

include "root" {
  path   = find_in_parent_folders()   # Tìm file terragrunt.hcl ở thư mục cha gần nhất
  expose = true                        # Cho phép truy cập locals của root config
}

terraform {
  source = "../../../../modules//eks"
}

inputs = {
  # Dùng locals từ root config (vì expose = true)
  environment      = include.root.locals.environment
  
  # Input riêng cho module này
  cluster_name     = "myapp-prod"
  cluster_version  = "1.28"
  instance_types   = ["t3.large", "t3.xlarge"]
  min_size         = 3
  max_size         = 10
}
```

### `find_in_parent_folders()` hoạt động thế nào?

```
Terragrunt tìm file "terragrunt.hcl" ở thư mục cha, ông, cố, ...
cho đến khi tìm thấy hoặc đến root filesystem.

Ví dụ: Đang ở live/us-east-1/prod/eks/
→ Tìm ở live/us-east-1/prod/       (không có)
→ Tìm ở live/us-east-1/            (không có)
→ Tìm ở live/                      (không có)
→ Tìm ở infra/                     (tìm thấy!)
```

---

## Dependency — Phụ thuộc — giữa modules

### Khai báo dependency

```hcl
# live/us-east-1/prod/eks/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../../modules//eks"
}

# Khai báo phụ thuộc vào module vpc
dependency "vpc" {
  config_path = "../vpc"   # Đường dẫn tương đối đến terragrunt.hcl của vpc

  # Mock outputs để plan mà không cần apply vpc trước
  mock_outputs = {
    vpc_id          = "vpc-mock-id"
    private_subnets = ["subnet-mock-1", "subnet-mock-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

# Khai báo phụ thuộc vào module security groups
dependency "security_groups" {
  config_path = "../security-groups"
  
  mock_outputs = {
    eks_sg_id = "sg-mock-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  # Dùng output từ module vpc
  vpc_id          = dependency.vpc.outputs.vpc_id
  subnet_ids      = dependency.vpc.outputs.private_subnet_ids
  
  # Dùng output từ module security_groups
  security_group_ids = [dependency.security_groups.outputs.eks_sg_id]

  cluster_name    = "myapp-prod"
  cluster_version = "1.28"
}
```

### Dependency graph — Đồ thị phụ thuộc

```
rds ─────────────────────────────────────────────────────┐
                                                          ↓
vpc ──────────────────────────────────────────────→ eks ──→ app (deployment)
                                                          ↑
security-groups ──────────────────────────────────────────┘

Terragrunt tự động tính toán thứ tự apply dựa trên dependency graph.
```

---

## Chạy nhiều modules cùng lúc

### `run-all` command

```bash
# Apply tất cả modules trong thư mục prod và sub-directories
cd live/us-east-1/prod
terragrunt run-all apply

# Terragrunt sẽ:
# 1. Scan tất cả thư mục có terragrunt.hcl
# 2. Xây dựng dependency graph
# 3. Apply theo đúng thứ tự (modules không phụ thuộc nhau chạy song song)

# Plan tất cả trước khi apply
terragrunt run-all plan

# Destroy theo thứ tự ngược lại
terragrunt run-all destroy

# Apply chỉ một số modules cụ thể
terragrunt run-all apply \
  --terragrunt-include-dir "vpc" \
  --terragrunt-include-dir "eks"

# Bỏ qua modules cụ thể
terragrunt run-all apply \
  --terragrunt-exclude-dir "rds"
```

### Output với run-all

```
Group 1
- Module live/us-east-1/prod/vpc             ← Không có dependency, chạy đầu tiên
- Module live/us-east-1/prod/security-groups ← Không có dependency, chạy song song

Group 2
- Module live/us-east-1/prod/eks             ← Phụ thuộc vpc + security-groups

Group 3
- Module live/us-east-1/prod/rds             ← Phụ thuộc vpc
- Module live/us-east-1/prod/app-infra       ← Phụ thuộc eks
```

---

## Khi nào nên dùng Terragrunt

### ✅ Nên dùng Terragrunt khi

1. **Nhiều môi trường (3+)** với cấu trúc tương tự
2. **Nhiều region hoặc nhiều account** cần quản lý
3. **Team lớn** cần DRY để giảm lỗi copy-paste
4. **Backend config lặp nhiều** — Terragrunt tạo tự động
5. **Cần orchestrate nhiều modules** có dependencies phức tạp
6. **Enterprise setup** với AWS Organizations, nhiều account

### ❌ Không nên dùng Terragrunt khi

1. **Dự án nhỏ** chỉ 1-2 môi trường
2. **Team chưa quen Terraform** — Học Terraform trước, Terragrunt sau
3. **Simple codebase** mà không có nhiều lặp lại
4. **Muốn tránh thêm dependency** — Thêm tool = thêm điều cần học và maintain
5. **Dùng Terraform Cloud** — Terraform Cloud có workspace management riêng

### Dấu hiệu bạn cần Terragrunt

```
❌ Copy-paste backend config cho mỗi môi trường mới
❌ Phải nhớ apply module A trước module B theo đúng thứ tự
❌ Code trong environments/dev/ và environments/prod/ gần như giống nhau
❌ Mỗi lần thêm môi trường mới mất > 1 giờ setup
```

---

## Câu hỏi phỏng vấn

### Q1: Terragrunt là gì và tại sao cần nó?

**Trả lời:**

Terragrunt là wrapper tool cho Terraform, giải quyết ba vấn đề chính:

1. **DRY backend config**: Thay vì copy-paste S3 backend config cho mỗi môi trường, định nghĩa một lần trong root `terragrunt.hcl` và tất cả modules kế thừa qua `include`.

2. **Cross-module dependencies**: Với `dependency` block, module eks có thể đọc output (`vpc_id`) từ module vpc mà không cần hardcode hay dùng data source phức tạp.

3. **Orchestration**: `terragrunt run-all apply` tự động tính dependency graph và apply theo đúng thứ tự, chạy song song các module không phụ thuộc nhau.

Khi nào nên dùng: Dự án có 3+ môi trường, backend config lặp nhiều, hoặc cần orchestrate nhiều modules có dependencies.

---

### Q2: Phan biệt Terragrunt dependency và Terraform depends_on?

**Trả lời:**

| | Terragrunt `dependency` | Terraform `depends_on` |
|--|------------------------|----------------------|
| Phạm vi | Giữa các thư mục module khác nhau | Trong cùng một Terraform config |
| Dùng cho | Cross-stack dependencies | Within-stack dependencies |
| Dữ liệu | Đọc outputs từ stack khác | Chỉ đảm bảo thứ tự |
| Ví dụ | eks đọc vpc_id từ stack vpc | EC2 tạo sau security group |

---

### Q3: Làm sao Terragrunt giúp quản lý multi-account?

**Trả lời:**

Với hierarchy config (account.hcl → region.hcl → env.hcl):

```
live/
├── account-dev/      ← account.hcl: account_id = "111222333"
│   └── us-east-1/
│       └── prod/
└── account-prod/     ← account.hcl: account_id = "444555666"
    └── us-east-1/
        └── prod/
```

Root `terragrunt.hcl` tự động đọc account_id từ `account.hcl` gần nhất trong hierarchy, tạo backend bucket name và IAM role ARN — Amazon Resource Name — phù hợp cho từng account.

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **Bắt đầu với Terraform thuần** — Hiểu rõ Terraform trước khi thêm Terragrunt
2. **Dùng `mock_outputs`** cho dependency khi plan — tránh phải apply modules không cần thiết
3. **Hierarchy config rõ ràng**: account.hcl → region.hcl → env.hcl — mỗi file một mục đích
4. **Pin Terraform và Terragrunt version** trong CI/CD để reproducible builds — Kết quả nhất quán
5. **Dùng `expose = true`** cho include khi cần access locals của parent config
6. **`run-all` cẩn thận ở production** — Luôn `plan` trước khi `apply`

---

**Tiếp theo:** [5-backend-per-env.md](./5-backend-per-env.md) — Backend riêng cho từng môi trường
