# Variables, Locals & Outputs — Biến, Biến Cục Bộ & Đầu Ra

> Variables là API — giao diện — của module Terraform. Locals là nơi tính toán trung gian. Outputs là cách chia sẻ thông tin ra bên ngoài. Ba thành phần này cùng nhau tạo ra Terraform configuration có thể tái sử dụng và bảo trì được.

---

## 📚 Mục Lục

1. [Variables — Biến Đầu Vào](#1-variables---biến-đầu-vào)
2. [Cách Truyền Giá Trị Vào Variable](#2-cách-truyền-giá-trị-vào-variable)
3. [Variable Validation — Kiểm Tra Giá Trị](#3-variable-validation---kiểm-tra-giá-trị)
4. [Sensitive Variables — Biến Nhạy Cảm](#4-sensitive-variables---biến-nhạy-cảm)
5. [Locals — Biến Cục Bộ](#5-locals---biến-cục-bộ)
6. [Outputs — Đầu Ra](#6-outputs---đầu-ra)
7. [Thứ Tự Ưu Tiên Variable](#7-thứ-tự-ưu-tiên-variable)
8. [Best Practices — Thực Hành Tốt Nhất](#8-best-practices---thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Variables — Biến Đầu Vào

Variable — Biến đầu vào — cho phép tham số hóa — parameterize — configuration, làm cho nó linh hoạt và tái sử dụng được cho nhiều môi trường khác nhau.

### Khai Báo Variable

```hcl
# variables.tf

# Variable đơn giản với default
variable "aws_region" {
  description = "AWS region để triển khai hạ tầng"
  type        = string
  default     = "ap-southeast-1"
}

# Variable bắt buộc (không có default)
variable "project" {
  description = "Tên dự án — dùng trong tên resource và tags"
  type        = string
  # Không có default → bắt buộc phải cung cấp
}

# Variable với type phức tạp
variable "vpc_config" {
  description = "Cấu hình VPC"
  type = object({
    cidr             = string
    public_subnets   = list(string)
    private_subnets  = list(string)
    enable_nat_gw    = bool
  })
  default = {
    cidr             = "10.0.0.0/16"
    public_subnets   = ["10.0.1.0/24", "10.0.2.0/24"]
    private_subnets  = ["10.0.11.0/24", "10.0.12.0/24"]
    enable_nat_gw    = false
  }
}

# Variable list
variable "availability_zones" {
  description = "Danh sách AZ — Availability Zones — Vùng khả dụng"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}

# Variable map
variable "instance_types" {
  description = "Instance type cho từng môi trường"
  type        = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}
```

### Dùng Variable

```hcl
# Trong resources, locals, other variables
resource "aws_vpc" "main" {
  cidr_block = var.vpc_config.cidr
}

resource "aws_instance" "web" {
  instance_type = var.instance_types[var.environment]  # Lookup map
  # hoặc
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")
}

output "region" {
  value = var.aws_region
}
```

---

## 2. Cách Truyền Giá Trị Vào Variable

### 2.1 File `.tfvars`

```hcl
# dev.tfvars — File biến cho môi trường dev
project     = "myapp"
environment = "dev"
aws_region  = "ap-southeast-1"

vpc_config = {
  cidr             = "10.0.0.0/16"
  public_subnets   = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets  = ["10.0.11.0/24", "10.0.12.0/24"]
  enable_nat_gw    = false  # dev không cần NAT Gateway — tốn tiền
}
```

```hcl
# prod.tfvars — File biến cho production
project     = "myapp"
environment = "prod"
aws_region  = "ap-southeast-1"

vpc_config = {
  cidr             = "10.1.0.0/16"
  public_subnets   = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
  private_subnets  = ["10.1.11.0/24", "10.1.12.0/24", "10.1.13.0/24"]
  enable_nat_gw    = true  # Production cần NAT Gateway
}
```

```bash
# Chạy với file tfvars cụ thể
terraform plan  -var-file=dev.tfvars
terraform apply -var-file=prod.tfvars
```

### 2.2 Auto-loaded Files — File Tự Động Load

Terraform **tự động** load các file sau (không cần `-var-file`):
- `terraform.tfvars`
- `terraform.tfvars.json`
- Bất kỳ file nào có đuôi `.auto.tfvars`
- Bất kỳ file nào có đuôi `.auto.tfvars.json`

```bash
# Cấu trúc tổ chức multi-environment
environments/
├── dev/
│   ├── main.tf → symlink hoặc module call
│   └── terraform.tfvars      # Tự động load
├── staging/
│   └── terraform.tfvars
└── prod/
    └── terraform.tfvars
```

### 2.3 Command Line — Dòng Lệnh

```bash
# Truyền trực tiếp (ưu tiên cao nhất sau env vars)
terraform apply -var="project=myapp" -var="environment=dev"

# Dùng cho giá trị nhạy cảm trong CI/CD
terraform apply -var="db_password=${DB_PASSWORD}"
```

### 2.4 Environment Variables — Biến Môi Trường

```bash
# Format: TF_VAR_<variable_name>
export TF_VAR_project="myapp"
export TF_VAR_environment="dev"
export TF_VAR_db_password="super-secret-password"

terraform apply
# Tự động lấy giá trị từ TF_VAR_* environment variables
```

### 2.5 Terraform Cloud / Enterprise Variable Sets

```
Trong Terraform Cloud UI hoặc API:
- Workspace variables
- Variable Sets — Bộ biến dùng chung cho nhiều workspace
```

---

## 3. Variable Validation — Kiểm Tra Giá Trị

```hcl
variable "environment" {
  description = "Môi trường triển khai"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment phải là một trong: dev, staging, prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Chỉ cho phép dùng t2.* hoặc t3.* instance types."
  }
}

variable "vpc_cidr" {
  description = "CIDR block cho VPC"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr phải là CIDR block hợp lệ (ví dụ: 10.0.0.0/16)."
  }
}

variable "instance_count" {
  description = "Số lượng EC2 instances"
  type        = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count phải từ 1 đến 10."
  }
}
```

**Lợi ích của validation:**
- Lỗi xuất hiện ngay khi `terraform plan` — fail fast — thất bại sớm
- Error message — thông báo lỗi — rõ ràng hơn API errors
- Tài liệu hóa constraints — ràng buộc — ngay trong code

---

## 4. Sensitive Variables — Biến Nhạy Cảm

```hcl
variable "db_password" {
  description = "Mật khẩu database"
  type        = string
  sensitive   = true  # Terraform sẽ che giá trị này trong output
}

variable "api_key" {
  description = "API key cho external service"
  type        = string
  sensitive   = true
}
```

```bash
# Khi có sensitive variable, Terraform che trong output:
# var.db_password
#   (sensitive value)

# Plan output:
# + db_password = (sensitive value)
```

```hcl
# Output cũng cần đánh dấu sensitive nếu chứa giá trị nhạy cảm
output "db_connection_string" {
  description = "Connection string cho database"
  value       = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.address}/mydb"
  sensitive   = true  # Bắt buộc nếu giá trị derived từ sensitive variable
}
```

### Thực Hành Tốt Với Secrets — Bí Mật

```hcl
# ❌ Không bao giờ hardcode secret trong .tf file
variable "db_password" {
  default = "mysecretpassword"  # NGUY HIỂM!
}

# ✅ Cách 1: Environment variable
# export TF_VAR_db_password="$(aws secretsmanager get-secret-value ...)"

# ✅ Cách 2: Đọc từ AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "/myapp/${var.environment}/db_password"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}

# ✅ Cách 3: Đọc từ SSM Parameter Store
data "aws_ssm_parameter" "db_password" {
  name            = "/myapp/${var.environment}/db_password"
  with_decryption = true  # Decrypt SecureString parameter
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

---

## 5. Locals — Biến Cục Bộ

**Locals — Biến cục bộ** là các giá trị được tính toán một lần và tái sử dụng trong module. Khác với variable, locals không thể override từ bên ngoài.

### Khai Báo và Dùng

```hcl
# locals.tf — hoặc locals block trong bất kỳ .tf file nào
locals {
  # Prefix — Tiền tố — dùng cho tất cả resource names
  prefix = "${var.project}-${var.environment}"

  # Common tags — Nhãn chung — áp dụng cho mọi resource
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Region      = var.aws_region
  }

  # Chọn instance type theo environment
  instance_type = lookup({
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }, var.environment, "t3.micro")

  # Logic phức tạp chỉ viết một lần
  is_production      = var.environment == "prod"
  enable_high_avail  = local.is_production
  db_backup_days     = local.is_production ? 30 : 7
  enable_deletion_protection = local.is_production
}

# Dùng trong resources
resource "aws_db_instance" "main" {
  identifier              = "${local.prefix}-db"
  instance_class          = local.is_production ? "db.t3.medium" : "db.t3.micro"
  backup_retention_period = local.db_backup_days
  deletion_protection     = local.enable_deletion_protection
  multi_az                = local.enable_high_avail

  tags = merge(local.common_tags, {
    Component = "database"
  })
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags = merge(local.common_tags, {
    Name      = "${local.prefix}-web"
    Component = "web"
  })
}
```

### Locals Cho Phép Biểu Thức Phức Tạp

```hcl
locals {
  # Xử lý list phức tạp
  all_cidr_blocks = concat(
    [var.vpc_cidr],
    var.additional_cidrs
  )

  # Tạo map từ list
  subnet_map = {
    for idx, subnet in var.subnet_cidrs :
    "subnet-${idx}" => subnet
  }

  # Conditional complex object
  database_config = local.is_production ? {
    instance_class    = "db.r6g.large"
    allocated_storage = 100
    multi_az          = true
  } : {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
  }
}
```

### Variable vs Local — Khi Nào Dùng Gì?

| Tiêu Chí | Variable | Local |
|----------|---------|-------|
| **Nguồn gốc** | User input — Đầu vào của người dùng | Computed — Tính toán nội bộ |
| **Override từ ngoài** | Có | Không |
| **Xuất hiện trong plan** | Có (làm tham số) | Không |
| **Dùng cho** | Cấu hình biến đổi theo môi trường | Giá trị tính toán trung gian |
| **Ví dụ** | `var.environment`, `var.region` | `local.prefix`, `local.common_tags` |

---

## 6. Outputs — Đầu Ra

**Outputs — Đầu ra** là cách Terraform export — xuất — thông tin về resource đã tạo. Có hai mục đích chính:
1. Hiển thị thông tin sau khi `apply` (cho người dùng)
2. Cho phép module khác tham chiếu (module composition — ghép module)

### Khai Báo Output

```hcl
# outputs.tf

# Output đơn giản
output "vpc_id" {
  description = "ID của VPC đã tạo"
  value       = aws_vpc.main.id
}

# Output với sensitive data
output "db_connection_string" {
  description = "Connection string database (ẩn trong log)"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

# Output với depends_on (hiếm dùng, chủ yếu cho provisioners)
output "deployment_complete" {
  description = "Marker output cho pipeline"
  value       = "true"
  depends_on  = [aws_instance.web]
}

# Output tập hợp
output "public_subnet_ids" {
  description = "Danh sách ID của public subnets"
  value       = [for subnet in aws_subnet.public : subnet.id]
}

output "instance_details" {
  description = "Chi tiết các web instances"
  value = {
    for name, instance in aws_instance.web :
    name => {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
    }
  }
}

# Output từ module (re-export — xuất lại)
output "alb_dns_name" {
  description = "DNS name của Application Load Balancer"
  value       = module.alb.dns_name
}
```

### Xem Output

```bash
# Xem tất cả outputs sau khi apply
terraform output

# Xem output cụ thể
terraform output vpc_id

# Output dưới dạng JSON — tiện cho scripts
terraform output -json

# Output raw (không có dấu nháy)
terraform output -raw vpc_id

# Dùng trong shell script
VPC_ID=$(terraform output -raw vpc_id)
echo "VPC ID: $VPC_ID"
```

### Module Outputs — Đầu Ra Module

```hcl
# Trong parent module, dùng output của child module
module "network" {
  source = "./modules/network"
  # ...
}

module "database" {
  source    = "./modules/database"
  vpc_id    = module.network.vpc_id         # Dùng output của module network
  subnet_ids = module.network.private_subnet_ids
}

module "application" {
  source      = "./modules/application"
  vpc_id      = module.network.vpc_id
  db_endpoint = module.database.endpoint   # Dùng output của module database
}

# Output cuối cùng ra ngoài
output "app_url" {
  value = "https://${module.application.alb_dns_name}"
}
```

---

## 7. Thứ Tự Ưu Tiên Variable

Khi cùng một variable được định nghĩa nhiều nơi, Terraform dùng thứ tự ưu tiên này (cao nhất đến thấp nhất):

```
1. -var hoặc -var-file trên command line       [Cao nhất]
2. *.auto.tfvars hoặc *.auto.tfvars.json       (theo alphabet)
3. terraform.tfvars.json
4. terraform.tfvars
5. TF_VAR_<name> environment variables
6. default trong variable block               [Thấp nhất]
```

```bash
# Ví dụ: environment được set ở nhiều chỗ
# terraform.tfvars: environment = "dev"
# TF_VAR_environment = "staging"
# -var flag: -var="environment=prod"

terraform apply -var="environment=prod"
# Kết quả: environment = "prod" (command line thắng)

# Nếu bỏ -var flag:
terraform apply
# Kết quả: environment = "dev" (từ terraform.tfvars)
```

---

## 8. Best Practices — Thực Hành Tốt Nhất

### Tổ Chức Variable

```hcl
# variables.tf — Nhóm variables theo chủ đề với separators

# ============================================================
# General — Cấu hình chung
# ============================================================
variable "project" { ... }
variable "environment" { ... }
variable "aws_region" { ... }

# ============================================================
# Networking — Mạng
# ============================================================
variable "vpc_cidr" { ... }
variable "availability_zones" { ... }

# ============================================================
# Compute — Tính toán
# ============================================================
variable "instance_type" { ... }
variable "instance_count" { ... }

# ============================================================
# Database — Cơ sở dữ liệu
# ============================================================
variable "db_instance_class" { ... }
variable "db_name" { ... }
```

### Đặt Tên Rõ Ràng

```hcl
# ❌ Tên mơ hồ
variable "type" { ... }
variable "size" { ... }
variable "count" { ... }

# ✅ Tên mô tả rõ
variable "ec2_instance_type" { ... }
variable "ebs_volume_size_gb" { ... }
variable "web_server_count" { ... }
```

### Description Đầy Đủ

```hcl
# ❌ Thiếu description hoặc quá ngắn
variable "retention" {
  type = number
}

# ✅ Description đầy đủ
variable "log_retention_days" {
  description = "Số ngày giữ lại CloudWatch logs. Giá trị hợp lệ: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653. Xem: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html"
  type        = number
  default     = 30
}
```

### Tổ Chức tfvars

```
environments/
├── common.tfvars           # Giá trị dùng chung mọi môi trường
├── dev.tfvars              # Override cho dev
├── staging.tfvars          # Override cho staging
└── prod.tfvars             # Override cho prod

# Chạy với nhiều file
terraform apply \
  -var-file=environments/common.tfvars \
  -var-file=environments/prod.tfvars
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Phân biệt variable, local, và output trong Terraform?**

> - **Variable:** Input từ bên ngoài — người dùng hoặc CI/CD. Có thể override bằng tfvars, env vars, command line.
> - **Local:** Giá trị tính toán nội bộ module, không thể override từ ngoài. Dùng để tránh lặp lại logic.
> - **Output:** Export thông tin ra ngoài module để hiển thị hoặc để module khác sử dụng.

**Q: Sensitive variable bảo vệ gì và không bảo vệ gì?**

> `sensitive = true` chỉ che giá trị trong **terminal output** (plan, apply) và **Terraform Cloud UI**. Nó **không** mã hóa trong state file — file trạng thái — và không ngăn được terraform state pull. State file vẫn chứa giá trị plaintext — văn bản thô. Để bảo vệ thực sự, cần mã hóa state backend và hạn chế quyền truy cập vào state. Tốt nhất là không để secret trong Terraform state bằng cách đọc từ Secrets Manager hoặc Vault.

**Q: Validation block trong variable hoạt động khi nào?**

> Validation chạy khi `terraform plan` hoặc `terraform apply`, ngay sau khi Terraform đọc các variable values. Nếu validation fail — thất bại — toàn bộ plan dừng lại với error message. Điều này giúp fail fast — thất bại sớm — trước khi gọi bất kỳ cloud API nào, giúp feedback loop — vòng phản hồi — nhanh hơn và error message rõ ràng hơn.

**Q: Khi nào nên dùng terraform.tfvars và khi nào nên dùng TF_VAR_?**

> `terraform.tfvars` — cho non-sensitive configs — cấu hình không nhạy cảm, commit vào git. `TF_VAR_` environment variables — cho secrets (passwords, API keys), không bao giờ lưu vào file. Trong CI/CD, set secrets qua environment variables của pipeline (GitHub Actions secrets, GitLab CI variables), còn các configs thông thường thì đọc từ `.tfvars` file trong repo.

---

## 📋 Checklist

- [ ] Khai báo variable với type, description, và default hợp lý
- [ ] Thêm validation cho variables quan trọng
- [ ] Đánh dấu sensitive = true cho secrets
- [ ] Dùng locals để tránh lặp lại logic phức tạp
- [ ] Khai báo output đầy đủ với description
- [ ] Hiểu thứ tự ưu tiên của variable sources

---

## 🔗 Liên Kết

- ← [3. Providers & Resources](./3-providers-resources.md)
- → [5. Lifecycle: init, plan, apply, destroy](./5-lifecycle.md)

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
