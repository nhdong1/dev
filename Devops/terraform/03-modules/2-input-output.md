# 2 — Variables, Outputs và Type Constraints

> Variables — Biến đầu vào — và Outputs — Đầu ra — tạo nên interface của module.
> Type Constraints — Ràng buộc kiểu — và Validations — Xác thực — giúp module fail-fast
> khi nhận input sai thay vì tạo ra hạ tầng sai.

---

## 📥 Input Variables — Biến Đầu Vào

### Cú Pháp Đầy Đủ

```hcl
variable "tên_biến" {
  description = "Giải thích biến này dùng để làm gì"
  type        = kiểu_dữ_liệu
  default     = giá_trị_mặc_định  # Nếu không có default → bắt buộc phải truyền
  sensitive   = true               # Ẩn giá trị khỏi logs và output
  nullable    = false              # Không cho phép truyền null

  validation {
    condition     = biểu_thức_boolean
    error_message = "Thông báo lỗi khi validation thất bại."
  }
}
```

### Các Kiểu Dữ Liệu Cơ Bản (Primitive Types)

```hcl
# string — Chuỗi ký tự
variable "region" {
  description = "AWS region để triển khai tài nguyên"
  type        = string
  default     = "us-east-1"
}

# number — Số (int hoặc float)
variable "instance_count" {
  description = "Số lượng EC2 instances cần tạo"
  type        = number
  default     = 2
}

# bool — Giá trị logic đúng/sai
variable "enable_monitoring" {
  description = "Bật CloudWatch detailed monitoring — Giám sát chi tiết"
  type        = bool
  default     = true
}
```

---

## 🔷 Type Constraints — Ràng Buộc Kiểu Nâng Cao

### Collection Types — Kiểu Tập Hợp

```hcl
# list(type) — Danh sách có thứ tự
variable "availability_zones" {
  description = "Danh sách AZs — Availability Zones — để triển khai"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

# map(type) — Bản đồ key-value
variable "tags" {
  description = "Tags áp dụng lên tất cả tài nguyên"
  type        = map(string)
  default     = {}
}

# set(type) — Tập hợp không có thứ tự, không trùng lặp
variable "allowed_cidr_blocks" {
  description = "Danh sách CIDR blocks được phép truy cập"
  type        = set(string)
  default     = []
}
```

### Structural Types — Kiểu Cấu Trúc

```hcl
# object — Đối tượng với các trường cố định
variable "database_config" {
  description = "Cấu hình database"
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    engine_version    = string
  })
  default = {
    instance_class    = "db.t3.medium"
    allocated_storage = 20
    multi_az          = false
    engine_version    = "14.9"
  }
}

# Dùng trong code:
resource "aws_db_instance" "this" {
  instance_class    = var.database_config.instance_class
  allocated_storage = var.database_config.allocated_storage
  multi_az          = var.database_config.multi_az
}
```

```hcl
# tuple — Bộ giá trị có kiểu khác nhau (ít dùng)
variable "subnet_config" {
  description = "[cidr, is_public]"
  type        = tuple([string, bool])
}

# list(object) — Danh sách đối tượng (phổ biến nhất)
variable "subnet_configs" {
  description = "Danh sách cấu hình cho từng subnet"
  type = list(object({
    cidr_block        = string
    availability_zone = string
    is_public         = bool
  }))
  default = []
}

# Dùng với for_each:
resource "aws_subnet" "this" {
  for_each = { for s in var.subnet_configs : s.cidr_block => s }

  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone
  map_public_ip_on_launch = each.value.is_public
}
```

---

## ✅ Validations — Xác Thực Đầu Vào

### Validation Cơ Bản

```hcl
variable "environment" {
  description = "Tên môi trường triển khai"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment phải là một trong: dev, staging, prod."
  }
}
```

### Validation Phức Tạp

```hcl
variable "vpc_cidr" {
  description = "CIDR block cho VPC"
  type        = string

  validation {
    # can() trả về true nếu biểu thức không lỗi
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr phải là CIDR IPv4 hợp lệ (ví dụ: 10.0.0.0/16)."
  }

  validation {
    # Chỉ cho phép dải IP private (RFC 1918)
    condition = anytrue([
      startswith(var.vpc_cidr, "10."),
      startswith(var.vpc_cidr, "172.16."),
      startswith(var.vpc_cidr, "192.168."),
    ])
    error_message = "vpc_cidr nên dùng dải IP private (RFC 1918): 10.x, 172.16.x, hoặc 192.168.x."
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 100
    error_message = "instance_count phải từ 1 đến 100."
  }
}

variable "name" {
  type = string

  validation {
    # regex() throw error nếu không khớp — can() bắt lỗi thành false
    condition     = can(regex("^[a-z][a-z0-9-]{2,29}$", var.name))
    error_message = "name phải: bắt đầu bằng chữ thường, chỉ chứa a-z, 0-9, dấu '-', dài 3-30 ký tự."
  }
}
```

---

## 🔒 Sensitive Variables — Biến Nhạy Cảm

```hcl
variable "database_password" {
  description = "Mật khẩu master của database"
  type        = string
  sensitive   = true  # Ẩn khỏi terraform plan/apply output và logs
}

variable "api_key" {
  description = "API key cho dịch vụ bên ngoài"
  type        = string
  sensitive   = true
}
```

```hcl
# ⚠️ Lưu ý: sensitive variable vẫn lưu plain text trong state file!
# Để bảo mật thực sự, dùng:
# - AWS Secrets Manager
# - HashiCorp Vault
# - Biến môi trường TF_VAR_*

# Cách truyền qua biến môi trường (không cần tfvars):
# export TF_VAR_database_password="my-secret-password"
# terraform apply
```

---

## 📤 Output Variables — Biến Đầu Ra

### Cú Pháp Cơ Bản

```hcl
output "tên_output" {
  description = "Giải thích output này chứa gì"
  value       = biểu_thức
  sensitive   = false   # true nếu output chứa dữ liệu nhạy cảm
  depends_on  = []      # Hiếm khi cần thiết
}
```

### Ví Dụ Thực Tế

```hcl
# outputs.tf của module vpc

output "vpc_id" {
  description = "ID của VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "Danh sách IDs của public subnets, theo thứ tự AZ"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Danh sách IDs của private subnets, theo thứ tự AZ"
  value       = aws_subnet.private[*].id
}

# Output map — Bản đồ theo tên AZ
output "private_subnet_by_az" {
  description = "Map từ tên AZ sang subnet ID"
  value = {
    for idx, subnet in aws_subnet.private :
    subnet.availability_zone => subnet.id
  }
}

# Output sensitive — Đầu ra nhạy cảm
output "db_connection_string" {
  description = "Connection string để kết nối database (nhạy cảm)"
  value       = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.this.endpoint}/${var.db_name}"
  sensitive   = true
}
```

### Lấy Output Từ Module Khác

```hcl
# Root module gọi hai module con
module "vpc" {
  source = "../../modules/networking/vpc"
  # ...
}

module "eks" {
  source = "../../modules/compute/eks-cluster"

  # Truyền output của module vpc vào module eks
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
}

# Root module expose output lên trên
output "cluster_endpoint" {
  description = "Endpoint của EKS cluster — Kubernetes cluster"
  value       = module.eks.cluster_endpoint
}
```

---

## 🎯 Best Practices — Thực Hành Tốt Nhất

### Đặt Tên Variables

```hcl
# ✅ Tên rõ ràng, dùng snake_case
variable "vpc_cidr_block" { }
variable "enable_nat_gateway" { }
variable "private_subnet_cidrs" { }

# ❌ Tên mơ hồ, viết tắt khó hiểu
variable "cidr" { }
variable "nat" { }
variable "prv_sub" { }
```

### Nhóm Related Variables Bằng Object

```hcl
# ❌ Nhiều variables riêng lẻ → interface lộn xộn
variable "db_instance_class" { type = string }
variable "db_allocated_storage" { type = number }
variable "db_backup_retention" { type = number }
variable "db_multi_az" { type = bool }

# ✅ Nhóm thành object → interface gọn hơn
variable "database" {
  description = "Cấu hình database"
  type = object({
    instance_class    = string
    allocated_storage = number
    backup_retention  = number
    multi_az          = bool
  })
}
```

### Optional Attributes Với Default (Terraform >= 1.3)

```hcl
variable "database" {
  description = "Cấu hình database"
  type = object({
    instance_class    = string
    allocated_storage = optional(number, 20)    # Tùy chọn, mặc định 20
    backup_retention  = optional(number, 7)     # Tùy chọn, mặc định 7 ngày
    multi_az          = optional(bool, false)   # Tùy chọn, mặc định false
  })
}

# Caller chỉ cần truyền instance_class:
module "db" {
  source = "./modules/rds"
  database = {
    instance_class = "db.t3.large"
    # Các trường khác dùng default
  }
}
```

---

## 🚦 Variable Precedence — Thứ Tự Ưu Tiên Biến

```
Thứ tự ưu tiên (cao → thấp):

1. -var "name=value"             Cao nhất — command line flag
2. -var-file="file.tfvars"       Command line var file
3. *.auto.tfvars.json            Tự động tải theo thứ tự bảng chữ cái
4. *.auto.tfvars                 Tự động tải theo thứ tự bảng chữ cái
5. terraform.tfvars.json         Tự động tải (nếu tồn tại)
6. terraform.tfvars              Tự động tải (nếu tồn tại)
7. TF_VAR_name                   Biến môi trường
8. default trong variable block  Thấp nhất
```

```bash
# Ví dụ thực tế:
terraform apply \
  -var-file="environments/prod.tfvars" \
  -var="instance_count=5"

# Hoặc dùng biến môi trường:
export TF_VAR_database_password="$(aws secretsmanager get-secret-value ...)"
terraform apply
```

---

## 📋 Checklist Variables & Outputs

```
Variables:
  [ ] Mọi variable có description rõ ràng
  [ ] Variables quan trọng có validation
  [ ] Variables nhạy cảm đánh dấu sensitive = true
  [ ] Không hard-code giá trị mặc định là production-specific
  [ ] Dùng type constraints (không dùng type = any trừ khi thực sự cần)

Outputs:
  [ ] Mọi output có description
  [ ] Expose đủ thông tin để caller dùng tiếp
  [ ] Outputs nhạy cảm đánh dấu sensitive = true
  [ ] Không expose internal resource attributes không cần thiết

Tổng thể:
  [ ] Interface của module minimal nhưng đủ dùng
  [ ] Tránh "leaky abstraction" — không để caller phải biết
      quá nhiều về cách module implement bên trong
```

---

## 💡 Ví Dụ Module EKS Node Group Đầy Đủ

```hcl
# variables.tf

variable "cluster_name" {
  description = "Tên EKS cluster — Kubernetes cluster mà node group này thuộc về"
  type        = string
}

variable "node_group_name" {
  description = "Tên của node group"
  type        = string
}

variable "node_role_arn" {
  description = "ARN — Amazon Resource Name — của IAM role cho worker nodes"
  type        = string
}

variable "subnet_ids" {
  description = "Danh sách subnet IDs để chạy worker nodes"
  type        = list(string)
}

variable "scaling_config" {
  description = "Cấu hình auto-scaling — Tự động mở rộng — cho node group"
  type = object({
    desired_size = number
    min_size     = number
    max_size     = number
  })
  default = {
    desired_size = 2
    min_size     = 1
    max_size     = 5
  }

  validation {
    condition     = var.scaling_config.min_size <= var.scaling_config.desired_size
    error_message = "desired_size phải >= min_size."
  }

  validation {
    condition     = var.scaling_config.desired_size <= var.scaling_config.max_size
    error_message = "desired_size phải <= max_size."
  }
}

variable "instance_types" {
  description = "Danh sách EC2 instance types cho worker nodes"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "capacity_type" {
  description = "ON_DEMAND hoặc SPOT — Spot instances giúp tiết kiệm chi phí"
  type        = string
  default     = "ON_DEMAND"

  validation {
    condition     = contains(["ON_DEMAND", "SPOT"], var.capacity_type)
    error_message = "capacity_type phải là ON_DEMAND hoặc SPOT."
  }
}

variable "labels" {
  description = "Kubernetes labels áp dụng lên nodes"
  type        = map(string)
  default     = {}
}

variable "tags" {
  description = "AWS resource tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# outputs.tf

output "node_group_id" {
  description = "ID của EKS node group"
  value       = aws_eks_node_group.this.id
}

output "node_group_arn" {
  description = "ARN của EKS node group"
  value       = aws_eks_node_group.this.arn
}

output "node_group_status" {
  description = "Trạng thái hiện tại của node group"
  value       = aws_eks_node_group.this.status
}

output "autoscaling_group_names" {
  description = "Danh sách tên Auto Scaling Groups — Nhóm tự động mở rộng — của node group"
  value       = aws_eks_node_group.this.resources[*].autoscaling_groups[*].name
}
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Tiếp Theo:** [3-module-registry.md](3-module-registry.md) — Terraform Registry public & private
