# .tfvars Management — Quản Lý File Biến Theo Môi Trường

> `.tfvars` files là cách Terraform nhận giá trị cụ thể cho từng môi trường mà không cần thay đổi code. Quản lý đúng giúp code sạch, an toàn; quản lý sai có thể lộ secrets lên git.

---

## 📚 Mục Lục

1. [.tfvars là gì?](#tfvars-là-gì)
2. [Cách Terraform load .tfvars](#cách-terraform-load-tfvars)
3. [Cấu trúc tfvars theo môi trường](#cấu-trúc-tfvars-theo-môi-trường)
4. [Quản lý secrets trong tfvars](#quản-lý-secrets-trong-tfvars)
5. [Variable validation — Xác thực biến](#variable-validation-xác-thực-biến)
6. [Patterns thực tế](#patterns-thực-tế)
7. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## .tfvars là gì?

`.tfvars` — Variable Files — là các file chứa giá trị cụ thể cho Terraform variables. Chúng tách **định nghĩa variable** (trong `variables.tf`) khỏi **giá trị cụ thể** (trong `.tfvars`).

### Ví dụ cơ bản

**`variables.tf` — Định nghĩa (không thay đổi theo môi trường):**
```hcl
variable "environment" {
  description = "Tên môi trường triển khai"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "min_capacity" {
  description = "Số lượng instance tối thiểu"
  type        = number
}

variable "enable_deletion_protection" {
  description = "Bảo vệ không bị xóa nhầm"
  type        = bool
  default     = false
}
```

**`dev.tfvars` — Giá trị cụ thể cho dev:**
```hcl
environment                = "dev"
instance_type              = "t3.micro"
min_capacity               = 1
enable_deletion_protection = false
```

**`prod.tfvars` — Giá trị cụ thể cho prod:**
```hcl
environment                = "prod"
instance_type              = "t3.large"
min_capacity               = 3
enable_deletion_protection = true
```

---

## Cách Terraform load .tfvars

### Thứ tự load (từ ưu tiên thấp đến cao)

Terraform load variables theo thứ tự sau, giá trị sau ghi đè giá trị trước:

```
1. Default values trong variables.tf            (ưu tiên thấp nhất)
2. terraform.tfvars                              (tự động load nếu tồn tại)
3. terraform.tfvars.json                         (tự động load nếu tồn tại)
4. *.auto.tfvars và *.auto.tfvars.json           (tự động load theo alphabet)
5. -var-file="file.tfvars" (truyền qua CLI)
6. -var "key=value" (truyền qua CLI)             (ưu tiên cao nhất)
```

### Cách truyền file tfvars

```bash
# Tự động: Terraform tự load terraform.tfvars trong cùng thư mục
terraform apply

# Thủ công: Chỉ định file cụ thể
terraform apply -var-file="dev.tfvars"

# Nhiều file (giá trị sau ghi đè giá trị trước)
terraform apply \
  -var-file="common.tfvars" \
  -var-file="dev.tfvars"

# Truyền biến trực tiếp (override tất cả)
terraform apply -var="instance_type=t3.small"
```

### File tự động load

```bash
# Các file này được Terraform tự động load
terraform.tfvars
terraform.tfvars.json
something.auto.tfvars
something.auto.tfvars.json

# File này KHÔNG tự động load — phải dùng -var-file
dev.tfvars
prod.tfvars
environments/dev.tfvars
```

---

## Cấu trúc tfvars theo môi trường

### Cấu trúc 1: Flat — Phẳng (Đơn giản)

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars
```

Chạy lệnh:
```bash
# Dev
terraform apply -var-file="dev.tfvars"

# Prod
terraform apply -var-file="prod.tfvars"
```

**Ưu điểm:** Đơn giản, dễ hiểu  
**Nhược điểm:** Tất cả trong cùng state, dễ nhầm

---

### Cấu trúc 2: Per-environment directory — Thư mục riêng (Khuyến nghị)

```
project/
├── modules/
│   └── app/
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars   ← Tự động load khi cd vào dev/
    ├── staging/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── terraform.tfvars
```

Khi dùng `terraform.tfvars` trong từng thư mục, Terraform tự động load — không cần `-var-file`:

```bash
# Làm việc với dev
cd environments/dev
terraform apply   # Tự động load terraform.tfvars

# Làm việc với prod
cd environments/prod
terraform apply   # Tự động load terraform.tfvars của prod
```

---

### Cấu trúc 3: Shared + override — Dùng chung + ghi đè

```
project/
├── variables.tf
├── common.tfvars       ← Giá trị dùng chung tất cả môi trường
├── dev.tfvars          ← Ghi đè cho dev
├── staging.tfvars
└── prod.tfvars
```

**`common.tfvars` — Giá trị chung:**
```hcl
region          = "us-east-1"
project_name    = "myapp"
owner_team      = "platform"
terraform_version = "~> 1.6"
```

**`dev.tfvars` — Ghi đè cho dev:**
```hcl
environment   = "dev"
instance_type = "t3.micro"
min_capacity  = 1
```

**Chạy lệnh:**
```bash
terraform apply \
  -var-file="common.tfvars" \
  -var-file="dev.tfvars"    # dev.tfvars ghi đè common nếu trùng key
```

---

## Quản lý secrets trong tfvars

### ❌ TUYỆT ĐỐI KHÔNG làm

```hcl
# prod.tfvars — ĐỪNG BAO GIỜ commit file này lên git nếu chứa secrets!
db_password    = "supersecretpassword123"    # Lộ credentials!
api_key        = "sk-prod-abcdef123456"      # Lộ API key!
jwt_secret     = "my-jwt-signing-secret"
```

```bash
# .gitignore phải có:
*.tfvars          # Hoặc cụ thể hơn:
prod.tfvars
staging.tfvars
*.secret.tfvars
```

### ✅ Cách đúng 1: Dùng environment variables — Biến môi trường

```bash
# Terraform tự động đọc biến có prefix TF_VAR_
export TF_VAR_db_password="supersecret"
export TF_VAR_api_key="sk-prod-abc"

terraform apply   # Không cần truyền -var
```

### ✅ Cách đúng 2: Đọc từ AWS Secrets Manager — Trình Quản Lý Bí Mật AWS

```hcl
# main.tf
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "myapp/${var.environment}/db-credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

resource "aws_db_instance" "main" {
  username = local.db_creds.username
  password = local.db_creds.password
  # ...
}
```

### ✅ Cách đúng 3: Dùng SOPS — Secrets OPerationS — để encrypt file

```bash
# Encrypt file trước khi commit
sops --encrypt prod.tfvars > prod.sops.tfvars

# Commit file đã encrypt (an toàn để commit)
git add prod.sops.tfvars

# Decrypt khi cần dùng
sops --decrypt prod.sops.tfvars > prod.tfvars
terraform apply -var-file="prod.tfvars"
rm prod.tfvars   # Xóa ngay sau khi dùng
```

### ✅ Cách đúng 4: CI/CD variables — Biến môi trường CI/CD

Trong GitHub Actions:
```yaml
- name: Terraform Apply
  run: terraform apply -auto-approve
  env:
    TF_VAR_db_password: ${{ secrets.PROD_DB_PASSWORD }}
    TF_VAR_api_key: ${{ secrets.PROD_API_KEY }}
```

---

## Variable validation — Xác thực biến

Thêm validation để phát hiện sai sót sớm:

```hcl
variable "environment" {
  description = "Tên môi trường"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment phải là một trong: dev, staging, prod."
  }
}

variable "instance_type" {
  description = "EC2 Instance Type"
  type        = string

  validation {
    condition = can(regex("^(t3|t3a|m5|c5)\\.", var.instance_type))
    error_message = "Chỉ cho phép các family: t3, t3a, m5, c5."
  }
}

variable "min_capacity" {
  description = "Số lượng instance tối thiểu"
  type        = number

  validation {
    condition     = var.min_capacity >= 1 && var.min_capacity <= 100
    error_message = "min_capacity phải từ 1 đến 100."
  }
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks được phép truy cập"
  type        = list(string)

  validation {
    condition = alltrue([
      for cidr in var.allowed_cidr_blocks :
      can(cidrhost(cidr, 0))
    ])
    error_message = "Tất cả giá trị phải là CIDR block hợp lệ (ví dụ: 10.0.0.0/8)."
  }
}
```

---

## Patterns thực tế

### Pattern 1: Environment-specific tfvars với common defaults

```hcl
# variables.tf
variable "tags" {
  description = "Tags mặc định cho tất cả tài nguyên"
  type        = map(string)
  default     = {}
}

variable "extra_tags" {
  description = "Tags bổ sung theo môi trường"
  type        = map(string)
  default     = {}
}

locals {
  # Merge tags chung với tags riêng của môi trường
  common_tags = {
    ManagedBy   = "terraform"
    Project     = var.project_name
    Environment = var.environment
  }
  
  all_tags = merge(local.common_tags, var.extra_tags)
}
```

```hcl
# prod.tfvars
environment  = "prod"
project_name = "myapp"

extra_tags = {
  CostCenter    = "prod-budget-001"
  Criticality   = "high"
  BackupPolicy  = "daily"
  Compliance    = "pci-dss"    # PCI-DSS tag để audit
}
```

---

### Pattern 2: Feature flags — Cờ tính năng — theo môi trường

```hcl
# variables.tf
variable "features" {
  description = "Feature flags cho từng môi trường"
  type = object({
    enable_waf           = bool    # WAF — Web Application Firewall
    enable_cloudfront    = bool    # CDN — Content Delivery Network
    enable_backup        = bool
    enable_monitoring    = bool
    enable_alerting      = bool
  })
  default = {
    enable_waf        = false
    enable_cloudfront = false
    enable_backup     = false
    enable_monitoring = true
    enable_alerting   = false
  }
}
```

```hcl
# dev.tfvars — Tắt các tính năng tốn kém
features = {
  enable_waf        = false
  enable_cloudfront = false
  enable_backup     = false
  enable_monitoring = true
  enable_alerting   = false
}
```

```hcl
# prod.tfvars — Bật tất cả
features = {
  enable_waf        = true
  enable_cloudfront = true
  enable_backup     = true
  enable_monitoring = true
  enable_alerting   = true
}
```

```hcl
# main.tf — Dùng feature flags
resource "aws_wafv2_web_acl" "main" {
  count = var.features.enable_waf ? 1 : 0
  # ...
}

resource "aws_cloudfront_distribution" "main" {
  count = var.features.enable_cloudfront ? 1 : 0
  # ...
}
```

---

### Pattern 3: Strict typing — Kiểu dữ liệu chặt chẽ — với objects

```hcl
# variables.tf
variable "database_config" {
  description = "Cấu hình database theo môi trường"
  type = object({
    instance_class      = string
    allocated_storage   = number
    multi_az            = bool
    backup_retention    = number   # Số ngày giữ backup
    deletion_protection = bool
    skip_final_snapshot = bool
  })
}

variable "network_config" {
  description = "Cấu hình mạng"
  type = object({
    vpc_cidr         = string
    azs              = list(string)
    private_subnets  = list(string)
    public_subnets   = list(string)
    enable_nat_gateway = bool
    single_nat_gateway = bool      # true = tiết kiệm chi phí nhưng single point of failure
  })
}
```

```hcl
# prod.tfvars
database_config = {
  instance_class      = "db.r5.large"
  allocated_storage   = 100
  multi_az            = true
  backup_retention    = 30       # Giữ backup 30 ngày
  deletion_protection = true
  skip_final_snapshot = false    # Luôn tạo snapshot khi xóa
}

network_config = {
  vpc_cidr           = "10.2.0.0/16"
  azs                = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets    = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
  public_subnets     = ["10.2.101.0/24", "10.2.102.0/24", "10.2.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = false     # Mỗi AZ một NAT Gateway cho HA
}
```

```hcl
# dev.tfvars
database_config = {
  instance_class      = "db.t3.small"
  allocated_storage   = 20
  multi_az            = false
  backup_retention    = 7
  deletion_protection = false
  skip_final_snapshot = true    # Không cần snapshot khi xóa dev
}

network_config = {
  vpc_cidr           = "10.0.0.0/16"
  azs                = ["us-east-1a", "us-east-1b"]
  private_subnets    = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets     = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = true     # Chỉ một NAT Gateway để tiết kiệm chi phí
}
```

---

## Câu hỏi phỏng vấn

### Q1: Làm sao quản lý secrets trong tfvars an toàn?

**Trả lời:**

Ba cách tiếp cận phổ biến theo mức độ bảo mật tăng dần:

1. **Environment variables** (`TF_VAR_*`): Đơn giản nhất, phù hợp với CI/CD systems
2. **AWS Secrets Manager / HashiCorp Vault**: Tốt nhất cho production, audit trail đầy đủ
3. **SOPS** — Secrets OPerationS: Encrypt file tfvars trước khi commit, có thể review trong git

Điểm chung: **Không bao giờ commit plaintext secrets** lên git. Thêm `.tfvars` vào `.gitignore`.

---

### Q2: Thứ tự ưu tiên của variable values trong Terraform là gì?

**Trả lời (từ thấp đến cao):**

```
1. default trong variables.tf
2. terraform.tfvars (tự động)
3. *.auto.tfvars (tự động, theo alphabet)
4. -var-file="file.tfvars" (CLI flag)
5. -var="key=value" (CLI flag, cao nhất)
```

Giá trị sau ghi đè giá trị trước.

---

### Q3: Tại sao nên dùng object type thay vì nhiều variables riêng lẻ?

**Trả lời:**

```hcl
# Không tốt: Nhiều variables rời rạc, dễ quên một số
variable "db_instance_class" { ... }
variable "db_allocated_storage" { ... }
variable "db_multi_az" { ... }
variable "db_backup_retention" { ... }

# Tốt hơn: Object gom lại, không thể thiếu field nào
variable "database_config" {
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    backup_retention  = number
  })
}
```

Lý do:
- **Type safety** — An toàn kiểu: Terraform báo lỗi ngay nếu thiếu field
- **Dễ đọc**: Cấu hình related gom lại một chỗ
- **Dễ truyền vào module**: Truyền một object thay vì 10 parameters

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **`.gitignore` phải có `*.tfvars`** — hoặc ít nhất là `prod.tfvars`
2. **Dùng object types** cho nhóm config liên quan, không dùng nhiều variables rời rạc
3. **Thêm validation** cho tất cả variables quan trọng
4. **Không để default = null** cho biến bắt buộc — bắt người dùng phải cung cấp giá trị
5. **Dùng description** rõ ràng cho mỗi variable — đây là documentation
6. **Commit `.tfvars.example`** thay vì `.tfvars` thật — làm tài liệu cho team
7. **CI/CD variables** cho secrets — không bao giờ hardcode trong file

---

**Tiếp theo:** [4-terragrunt-intro.md](./4-terragrunt-intro.md) — Terragrunt: Giải pháp DRY cho đa môi trường
