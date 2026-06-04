# 1 — Cấu Trúc Module Chuẩn (Module Structure)

> Module structure — Cấu trúc module — là cách tổ chức file và thư mục bên trong một module Terraform.
> Cấu trúc chuẩn giúp module dễ đọc, dễ maintain và dễ được người khác sử dụng.

---

## 📁 Cấu Trúc Tối Thiểu

```
modules/vpc/
├── main.tf          ← Tài nguyên chính (bắt buộc)
├── variables.tf     ← Khai báo input variables (bắt buộc)
├── outputs.tf       ← Khai báo outputs (bắt buộc)
└── README.md        ← Tài liệu usage (nên có)
```

Đây là cấu trúc tối thiểu để module hoạt động được. Mọi thứ còn lại là tùy chọn nhưng được khuyến khích.

---

## 📁 Cấu Trúc Đầy Đủ (Khuyến Nghị)

```
modules/vpc/
├── main.tf              ← Resources chính
├── variables.tf         ← Input variables
├── outputs.tf           ← Outputs
├── versions.tf          ← Terraform & provider version constraints
├── locals.tf            ← Local values — Giá trị cục bộ
├── data.tf              ← Data sources — Nguồn dữ liệu
├── README.md            ← Tài liệu, usage examples
├── CHANGELOG.md         ← Lịch sử thay đổi theo version
└── examples/
    ├── basic/           ← Ví dụ đơn giản
    │   ├── main.tf
    │   └── outputs.tf
    └── complete/        ← Ví dụ đầy đủ tất cả tính năng
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 📄 Nội Dung Từng File

### `main.tf` — Tài Nguyên Chính

File quan trọng nhất. Chứa tất cả `resource` và `module` blocks.

```hcl
# modules/vpc/main.tf

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = true

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-vpc"
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
      Tier = "public"
    }
  )
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-private-${count.index + 1}"
      Tier = "private"
    }
  )
}

resource "aws_internet_gateway" "this" {
  count = var.create_internet_gateway ? 1 : 0

  vpc_id = aws_vpc.this.id

  tags = merge(var.tags, { Name = "${var.name}-igw" })
}
```

---

### `variables.tf` — Biến Đầu Vào

Khai báo tất cả input variables. **Không** đặt giá trị ở đây — chỉ khai báo.

```hcl
# modules/vpc/variables.tf

variable "name" {
  description = "Tên prefix dùng cho tất cả tài nguyên trong module này"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block — Dải địa chỉ IP — của VPC (ví dụ: 10.0.0.0/16)"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr phải là CIDR block hợp lệ (ví dụ: 10.0.0.0/16)."
  }
}

variable "public_subnet_cidrs" {
  description = "Danh sách CIDR blocks cho public subnets"
  type        = list(string)
  default     = []
}

variable "private_subnet_cidrs" {
  description = "Danh sách CIDR blocks cho private subnets"
  type        = list(string)
  default     = []
}

variable "availability_zones" {
  description = "Danh sách Availability Zones — Vùng sẵn sàng — để triển khai subnets"
  type        = list(string)
}

variable "enable_dns_hostnames" {
  description = "Bật DNS hostnames trong VPC"
  type        = bool
  default     = true
}

variable "create_internet_gateway" {
  description = "Tạo Internet Gateway — Cổng Internet — cho VPC"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Map các tags áp dụng lên tất cả tài nguyên"
  type        = map(string)
  default     = {}
}
```

---

### `outputs.tf` — Đầu Ra

Expose thông tin mà caller cần dùng sau khi module tạo tài nguyên.

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID của VPC vừa được tạo"
  value       = aws_vpc.this.id
}

output "vpc_cidr_block" {
  description = "CIDR block của VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Danh sách IDs của public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Danh sách IDs của private subnets"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "ID của Internet Gateway (null nếu không tạo)"
  value       = var.create_internet_gateway ? aws_internet_gateway.this[0].id : null
}
```

---

### `versions.tf` — Ràng Buộc Phiên Bản

Khai báo phiên bản Terraform và providers cần thiết.

```hcl
# modules/vpc/versions.tf

terraform {
  required_version = ">= 1.3.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}
```

> **Tại sao cần file này?**
> Đảm bảo module không bị dùng với phiên bản Terraform quá cũ.
> Dùng `>=` (không dùng `=` cứng) để tránh xung đột với caller.

---

### `locals.tf` — Giá Trị Cục Bộ (Local Values)

Dùng để tính toán hoặc chuẩn hóa giá trị trong nội bộ module.

```hcl
# modules/vpc/locals.tf

locals {
  # Tính số lượng AZs — Availability Zones — được yêu cầu
  az_count = max(
    length(var.public_subnet_cidrs),
    length(var.private_subnet_cidrs)
  )

  # Tags chung áp dụng lên mọi tài nguyên
  common_tags = merge(
    var.tags,
    {
      ManagedBy = "terraform"
      Module    = "vpc"
    }
  )

  # Tên base nhất quán
  name_prefix = lower(replace(var.name, " ", "-"))
}
```

---

### `data.tf` — Data Sources

Truy vấn thông tin từ cloud provider mà không tạo tài nguyên mới.

```hcl
# modules/vpc/data.tf

# Lấy danh sách AZs — Availability Zones — có sẵn trong region
data "aws_availability_zones" "available" {
  state = "available"
}

# Lấy thông tin AMI — Amazon Machine Image — mới nhất
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

---

## 🏗️ Ví Dụ Module Hoàn Chỉnh: RDS PostgreSQL

```
modules/rds-postgres/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── locals.tf
└── README.md
```

```hcl
# modules/rds-postgres/main.tf

resource "aws_db_subnet_group" "this" {
  name       = "${local.identifier}-subnet-group"
  subnet_ids = var.subnet_ids

  tags = local.common_tags
}

resource "aws_db_instance" "this" {
  identifier        = local.identifier
  engine            = "postgres"
  engine_version    = var.engine_version
  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage

  db_name  = var.database_name
  username = var.master_username
  password = var.master_password  # Nên dùng Secrets Manager — Trình quản lý bí mật

  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = var.security_group_ids

  # Cài đặt backup — Sao lưu
  backup_retention_period = var.backup_retention_days
  backup_window           = "03:00-04:00"

  # Bảo vệ xoá tình cờ
  deletion_protection = var.deletion_protection
  skip_final_snapshot = var.skip_final_snapshot

  tags = local.common_tags
}
```

---

## 📂 Cấu Trúc Repository Nhiều Modules

```
infrastructure/
├── modules/                        ← Tất cả shared modules
│   ├── networking/
│   │   ├── vpc/                    ← modules/networking/vpc
│   │   ├── security-groups/
│   │   └── load-balancer/
│   ├── compute/
│   │   ├── ec2-autoscaling/
│   │   └── ecs-service/
│   ├── database/
│   │   ├── rds-postgres/
│   │   └── elasticache-redis/
│   └── platform/
│       ├── eks-cluster/
│       └── s3-static-site/
│
├── environments/                   ← Root modules gọi shared modules
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
│
└── examples/                       ← Ví dụ cách dùng từng module
    ├── vpc-basic/
    └── full-stack/
```

---

## 📝 README.md Module Chuẩn

README tốt là chìa khóa để người khác dùng module đúng cách.

```markdown
# Module: VPC — Virtual Private Cloud — Mạng Riêng Ảo

Tạo VPC trên AWS với public và private subnets, Internet Gateway.

## Usage — Cách Dùng

```hcl
module "vpc" {
  source = "../../modules/networking/vpc"

  name    = "my-app"
  vpc_cidr = "10.0.0.0/16"

  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]

  tags = {
    Environment = "production"
    Project     = "my-app"
  }
}
```

## Requirements — Yêu Cầu

| Tên      | Phiên Bản |
|----------|-----------|
| terraform | >= 1.3.0 |
| aws      | >= 4.0    |

## Inputs — Đầu Vào

| Tên                  | Mô Tả                              | Kiểu           | Mặc Định | Bắt Buộc |
|----------------------|------------------------------------|----------------|----------|----------|
| name                 | Tên prefix cho tài nguyên          | string         | —        | ✅       |
| vpc_cidr             | CIDR block của VPC                 | string         | —        | ✅       |
| availability_zones   | Danh sách AZs                      | list(string)   | —        | ✅       |
| public_subnet_cidrs  | CIDRs cho public subnets           | list(string)   | []       | ❌       |
| private_subnet_cidrs | CIDRs cho private subnets          | list(string)   | []       | ❌       |
| tags                 | Tags áp dụng lên tài nguyên        | map(string)    | {}       | ❌       |

## Outputs — Đầu Ra

| Tên                  | Mô Tả                              |
|----------------------|------------------------------------|
| vpc_id               | ID của VPC                         |
| public_subnet_ids    | Danh sách IDs public subnets       |
| private_subnet_ids   | Danh sách IDs private subnets      |
```
```

---

## ✅ Checklist Cấu Trúc Module

```
File cơ bản:
  [ ] main.tf — chứa resources
  [ ] variables.tf — khai báo inputs
  [ ] outputs.tf — khai báo outputs
  [ ] README.md — hướng dẫn sử dụng

File nâng cao (nên có):
  [ ] versions.tf — terraform & provider version constraints
  [ ] locals.tf — tính toán nội bộ
  [ ] data.tf — data sources nếu cần

Tài liệu:
  [ ] README có usage example hoạt động được
  [ ] Variables có description đủ rõ
  [ ] Outputs có description đủ rõ
  [ ] CHANGELOG.md cho tracked versions

Examples:
  [ ] Có ít nhất 1 example đơn giản
  [ ] Example có thể chạy được độc lập
```

---

## 💡 Tips Thực Tế

```
1. Đặt tên file nhất quán: main.tf, variables.tf, outputs.tf — mọi người
   đều biết tìm gì ở đâu.

2. Một module = một thư mục: không chia module thành nhiều thư mục.

3. locals.tf tách riêng: khi module có nhiều local values, tách ra
   file riêng giúp main.tf gọn hơn.

4. examples/ rất quan trọng: caller dùng examples như documentation
   sống — luôn cập nhật khi module thay đổi.

5. Resource naming trong module: dùng "this" khi module tạo một
   resource chính (aws_vpc.this), dùng tên mô tả khi tạo nhiều
   (aws_subnet.public, aws_subnet.private).
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Tiếp Theo:** [2-input-output.md](2-input-output.md) — Variables, Outputs và Type Constraints
