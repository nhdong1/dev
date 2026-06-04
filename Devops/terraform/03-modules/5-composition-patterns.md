# 5 — Module Composition Patterns — Kiểu Mẫu Ghép Module

> Module composition — Ghép module — là cách tổ chức và kết hợp nhiều modules
> để xây dựng hạ tầng phức tạp từ các thành phần đơn giản.
> Ba pattern chính: Flat, Nested, và Wrapper.

---

## 🗺️ Tổng Quan Ba Pattern

```
┌─────────────────────────────────────────────────────────┐
│  Flat Pattern — Pattern Phẳng                           │
│  Root Module gọi trực tiếp tất cả modules con          │
│  root → [module_a, module_b, module_c, module_d]        │
├─────────────────────────────────────────────────────────┤
│  Nested Pattern — Pattern Lồng Nhau                     │
│  Module gọi module khác bên trong                       │
│  root → service_module → [vpc, eks, rds]               │
├─────────────────────────────────────────────────────────┤
│  Wrapper Pattern — Pattern Bao Ngoài                    │
│  Module mỏng bao ngoài module khác, thêm conventions   │
│  root → internal_wrapper → public_module               │
└─────────────────────────────────────────────────────────┘
```

---

## 🔷 Pattern 1: Flat Composition — Ghép Phẳng

### Mô Tả

Root module gọi trực tiếp tất cả child modules mà không có tầng nào ở giữa.

```
environments/prod/
└── main.tf            ← Gọi trực tiếp tất cả modules
    ├── module "vpc" { }
    ├── module "security_groups" { }
    ├── module "eks" { }
    ├── module "rds" { }
    └── module "monitoring" { }
```

### Code Ví Dụ

```hcl
# environments/prod/main.tf

module "vpc" {
  source  = "../../modules/networking/vpc"
  version = "2.1.0"

  name    = local.name
  cidr    = "10.0.0.0/16"
  azs     = data.aws_availability_zones.available.names

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  tags = local.common_tags
}

module "security_groups" {
  source = "../../modules/networking/security-groups"

  vpc_id      = module.vpc.vpc_id
  environment = var.environment
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.1"

  cluster_name = "${local.name}-cluster"
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids

  node_security_group_additional_rules = {
    ingress_from_alb = {
      type        = "ingress"
      from_port   = 8080
      to_port     = 8080
      protocol    = "tcp"
      source_security_group_id = module.security_groups.alb_sg_id
    }
  }
}

module "rds" {
  source = "../../modules/database/rds-postgres"

  identifier   = "${local.name}-db"
  subnet_ids   = module.vpc.database_subnet_ids
  sg_ids       = [module.security_groups.rds_sg_id]
  environment  = var.environment
}

module "monitoring" {
  source = "../../modules/platform/monitoring"

  cluster_name = module.eks.cluster_name
  rds_id       = module.rds.db_instance_id
  environment  = var.environment
}
```

### Ưu / Nhược Điểm

```
✅ Ưu điểm:
   - Dependencies rõ ràng — thấy ngay module nào dùng output của module nào
   - Dễ debug — terraform graph đơn giản
   - Linh hoạt — dễ thay thế từng module
   - Terraform plan nhanh hơn (ít tầng hơn)

❌ Nhược điểm:
   - Root module có thể dài và phức tạp
   - Lặp lại boilerplate — code lặp — giữa environments
   - Caller phải hiểu và truyền đúng outputs giữa modules
```

### Khi Nào Dùng Flat

```
✅ Dùng Flat khi:
   - Hạ tầng có ít modules (< 8-10 modules)
   - Team nhỏ, tất cả hiểu toàn bộ stack
   - Cần kiểm soát chi tiết từng resource
   - Đây là recommendation mặc định cho hầu hết projects
```

---

## 🔷 Pattern 2: Nested Composition — Ghép Lồng Nhau

### Mô Tả

Module gọi module khác bên trong, tạo ra các tầng abstraction.

```
environments/prod/
└── main.tf
    └── module "backend_service" {     ← Service module
        ├── module "vpc" { }           ← Infrastructure module
        ├── module "eks" { }
        └── module "rds" { }
        }
```

### Code Ví Dụ

```hcl
# modules/services/backend-api/main.tf
# Module này gọi infrastructure modules bên trong

terraform {
  required_providers {
    aws = { source = "hashicorp/aws" }
  }
}

# Gọi networking module bên trong service module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.service_name}-${var.environment}"
  cidr = var.vpc_cidr
  azs  = var.availability_zones

  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway   = true
  single_nat_gateway   = var.environment != "prod"

  tags = local.common_tags
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name = "${var.service_name}-${var.environment}"
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      instance_types = var.node_instance_types
      min_size       = var.min_nodes
      max_size       = var.max_nodes
      desired_size   = var.desired_nodes
    }
  }

  tags = local.common_tags
}

module "rds" {
  source = "../../../modules/database/rds-postgres"

  identifier = "${var.service_name}-${var.environment}"
  subnet_ids = module.vpc.database_subnets
  # ...
}
```

```hcl
# modules/services/backend-api/variables.tf

variable "service_name" {
  description = "Tên dịch vụ backend"
  type        = string
}

variable "environment" {
  description = "Môi trường (dev/staging/prod)"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block cho VPC của dịch vụ này"
  type        = string
  default     = "10.0.0.0/16"
}

variable "node_instance_types" {
  description = "EC2 instance types cho EKS nodes"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "min_nodes" {
  type    = number
  default = 1
}

variable "max_nodes" {
  type    = number
  default = 10
}

variable "desired_nodes" {
  type    = number
  default = 2
}
```

```hcl
# environments/prod/main.tf — Root module rất gọn

module "backend_api" {
  source = "../../modules/services/backend-api"

  service_name = "my-app"
  environment  = "prod"
  vpc_cidr     = "10.0.0.0/16"

  node_instance_types = ["m5.large"]
  min_nodes    = 3
  max_nodes    = 20
  desired_nodes = 5
}
```

### Ưu / Nhược Điểm

```
✅ Ưu điểm:
   - Root module rất đơn giản — dễ đọc cho người mới
   - Đóng gói hoàn chỉnh — caller không cần biết implementation details
   - DRY — Don't Repeat Yourself — giữa các environments

❌ Nhược điểm:
   - Debugging khó hơn — phải trace qua nhiều tầng
   - Khó customize một phần của nested module
   - terraform plan chậm hơn
   - Module coupling chặt — khó swap một module con
   - Khó test từng tầng riêng lẻ
```

### Giới Hạn Nesting — Terraform Limitations

```
⚠️ Terraform có một số hạn chế với nested modules:

1. Provider configuration:
   Nested modules không thể cấu hình providers riêng
   → Phải truyền provider từ root qua configuration_aliases

2. Count/for_each limitations:
   Không thể dùng count hoặc for_each trên nested modules
   với dynamic values từ module cha trong một số cases

3. Visibility:
   Resources trong nested module không hiển thị trực tiếp
   trong terraform state list của root → khó operate

Recommendation: Tối đa 2 tầng nesting
```

---

## 🔷 Pattern 3: Wrapper Pattern — Module Bao Ngoài

### Mô Tả

Wrapper module — Module bao ngoài — là một module mỏng bao ngoài module khác (thường là public module), thêm vào các conventions, defaults, và restrictions của tổ chức.

```
modules/
└── wrappers/
    └── aws-vpc/           ← Internal wrapper
        ├── main.tf        ← Gọi public module bên trong
        ├── variables.tf   ← Chỉ expose variables được phép
        └── outputs.tf     ← Chỉ expose outputs cần thiết
            │
            └── gọi → terraform-aws-modules/vpc/aws  (public module)
```

### Code Ví Dụ: VPC Wrapper

```hcl
# modules/wrappers/aws-vpc/main.tf

# Gọi public module nhưng enforce company defaults
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"   # Pinned — public module version được kiểm soát tập trung

  name = "${var.name}-${var.environment}"
  cidr = var.cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  # ===== Company defaults — Không expose ra ngoài =====

  # Bắt buộc bật DNS (company policy)
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Bắt buộc bật VPC Flow Logs (company security policy)
  enable_flow_log                      = true
  create_flow_log_cloudwatch_iam_role  = true
  create_flow_log_cloudwatch_log_group = true

  # Bắt buộc tag đầy đủ (company tagging policy)
  tags = merge(
    var.tags,
    {
      ManagedBy   = "terraform"
      Team        = var.team_name
      CostCenter  = var.cost_center
      Environment = var.environment
    }
  )

  # NAT Gateway strategy theo environment
  enable_nat_gateway = true
  single_nat_gateway = var.environment == "dev" || var.environment == "staging"
  one_nat_gateway_per_az = var.environment == "prod"
}
```

```hcl
# modules/wrappers/aws-vpc/variables.tf
# Chỉ expose những gì cần thiết — giảm cognitive load

variable "name" {
  description = "Tên project/service"
  type        = string
}

variable "environment" {
  description = "Môi trường (dev/staging/prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Chỉ chấp nhận: dev, staging, prod."
  }
}

variable "cidr" {
  description = "CIDR block cho VPC"
  type        = string
}

variable "availability_zones" {
  description = "Danh sách AZs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  type    = list(string)
  default = []
}

variable "public_subnet_cidrs" {
  type    = list(string)
  default = []
}

variable "team_name" {
  description = "Tên team sở hữu hạ tầng (dùng cho tagging)"
  type        = string
}

variable "cost_center" {
  description = "Cost center code cho billing allocation"
  type        = string
}

variable "tags" {
  description = "Tags bổ sung"
  type        = map(string)
  default     = {}
}

# ❌ Không expose: enable_flow_log, enable_dns_*, single_nat_gateway
# → Những thứ này là company policy, không cho teams override
```

```hcl
# modules/wrappers/aws-vpc/outputs.tf
# Expose những outputs cần thiết, format lại nếu cần

output "vpc_id" {
  description = "ID của VPC"
  value       = module.vpc.vpc_id
}

output "private_subnet_ids" {
  description = "IDs của private subnets"
  value       = module.vpc.private_subnets
}

output "public_subnet_ids" {
  description = "IDs của public subnets"
  value       = module.vpc.public_subnets
}

output "database_subnet_group" {
  description = "Tên DB subnet group"
  value       = module.vpc.database_subnet_group_name
}
```

### Dùng Wrapper Module

```hcl
# environments/prod/main.tf

module "vpc" {
  source = "../../modules/wrappers/aws-vpc"

  name        = "my-service"
  environment = "prod"
  cidr        = "10.0.0.0/16"
  team_name   = "platform-team"
  cost_center = "CC-1234"

  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnet_cidrs  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  # Không cần quan tâm đến:
  # - Flow logs (tự động bật)
  # - DNS settings (tự động bật)
  # - NAT Gateway mode (tự động theo environment)
  # - Required tags (tự động thêm)
}
```

### Ưu / Nhược Điểm

```
✅ Ưu điểm:
   - Enforce company standards — Áp đặt tiêu chuẩn công ty
   - Caller API đơn giản hơn module gốc
   - Dễ upgrade public module (chỉ 1 chỗ)
   - Không phải mỗi team tự cấu hình security/compliance settings
   - Giảm security misconfiguration

❌ Nhược điểm:
   - Thêm tầng abstraction — có thể gây confusion
   - Cần maintain wrapper khi public module thay đổi API
   - Có thể quá opinionated — hạn chế legitimate use cases
```

---

## 🔄 Kết Hợp Patterns: Real-World Architecture

```
Thực tế thường kết hợp cả ba patterns:

infrastructure/
├── modules/
│   ├── wrappers/              ← Wrapper Pattern
│   │   ├── aws-vpc/           (bao ngoài public VPC module)
│   │   ├── aws-eks/           (bao ngoài public EKS module)
│   │   └── aws-rds/           (bao ngoài public RDS module)
│   │
│   └── services/              ← Nested Pattern (tùy chọn)
│       └── kubernetes-platform/
│           ├── main.tf        (gọi wrapper modules)
│           ├── variables.tf
│           └── outputs.tf
│
└── environments/              ← Flat Pattern (root modules)
    ├── dev/
    │   └── main.tf            (gọi wrappers trực tiếp hoặc service modules)
    ├── staging/
    └── prod/
```

---

## ⚡ Pattern So Sánh Trực Tiếp

### Scenario: Deploy EKS + VPC + RDS

```hcl
# === Flat Pattern ===
# environments/prod/main.tf (40-80 dòng)

module "vpc" { source = "../../modules/wrappers/aws-vpc"; ... }
module "eks" { source = "../../modules/wrappers/aws-eks";
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  ...
}
module "rds" { source = "../../modules/wrappers/aws-rds";
  subnet_ids = module.vpc.private_subnet_ids
  ...
}
```

```hcl
# === Nested Pattern ===
# environments/prod/main.tf (10-20 dòng)

module "backend_stack" {
  source = "../../modules/services/backend-stack"
  environment = "prod"
  vpc_cidr    = "10.0.0.0/16"
  # ... ít params hơn nhiều
}
```

```hcl
# === Wrapper Pattern (được dùng trong cả hai) ===
# modules/wrappers/aws-vpc/main.tf

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"  # Public module
  version = "5.1.2"
  # ... enforce company defaults
}
```

---

## 🎯 Hướng Dẫn Chọn Pattern

```
Câu hỏi 1: Có dùng public modules không?
  → Có: Cân nhắc Wrapper Pattern để enforce standards

Câu hỏi 2: Root module có > 10 modules không?
  → Có: Cân nhắc Nested Pattern để nhóm theo service

Câu hỏi 3: Nhiều environments copy-paste logic không?
  → Có: Cân nhắc Nested Pattern hoặc Service Module

Câu hỏi 4: Team lớn, nhiều teams dùng chung modules?
  → Có: Wrapper Pattern giúp enforce standards

Câu hỏi 5: Cần flexibility cao, dễ debug?
  → Flat Pattern là lựa chọn tốt
```

| Tiêu Chí            | Flat      | Nested    | Wrapper   |
| ------------------- | --------- | --------- | --------- |
| Độ phức tạp code    | Trung bình | Thấp     | Trung bình|
| Khả năng debug      | Dễ        | Khó       | Trung bình|
| DRY giữa envs       | Thấp      | Cao       | Không     |
| Enforce standards   | Thấp      | Trung bình| Cao       |
| Flexibility         | Cao       | Thấp      | Trung bình|
| Dùng cho teams      | Nhỏ-vừa  | Vừa-lớn  | Lớn       |

---

## 🏗️ Refactoring Modules — Tái Cấu Trúc Module An Toàn

### Đổi Tên Resource Trong Module

```hcl
# Vấn đề: Đổi tên resource sẽ destroy + recreate
# Giải pháp: Dùng moved block — Khối di chuyển

moved {
  from = module.vpc.aws_subnet.public
  to   = module.vpc.aws_subnet.public_subnet
}

moved {
  from = module.old_vpc
  to   = module.vpc
}
```

### Di Chuyển Resource Ra Khỏi Module

```hcl
# Di chuyển resource từ module ra root (hoặc ngược lại)
moved {
  from = module.vpc.aws_internet_gateway.this[0]
  to   = aws_internet_gateway.main
}
```

### Tách Module Lớn Thành Nhiều Module Nhỏ

```
Quy trình an toàn:

1. Tạo module mới với resources tách ra
2. Thêm moved blocks trong module cũ pointing đến module mới
3. Chạy terraform plan — kiểm tra không có destroy
4. Apply
5. Xoá code cũ khỏi module gốc (sau khi confirm okay)
6. Xoá moved blocks (sau vài sprints — khi không còn rollback cần thiết)
```

---

## ✅ Checklist Composition Pattern

```
Trước khi chọn pattern:
  [ ] Đánh giá độ phức tạp của hạ tầng
  [ ] Xem xét team size và skill level
  [ ] Xác định cần enforce standards không

Khi implement:
  [ ] Flat: Kiểm tra root module không > 200-300 dòng
  [ ] Nested: Giới hạn tối đa 2-3 tầng nesting
  [ ] Wrapper: Không để wrapper phức tạp hơn module gốc

Khi refactor:
  [ ] Dùng moved blocks — không destroy rồi recreate
  [ ] Test với terraform plan trước khi apply
  [ ] Cập nhật CHANGELOG nếu đây là shared module
  [ ] Thông báo cho tất cả callers về breaking changes

Tổng thể:
  [ ] Module có single responsibility
  [ ] Dependencies rõ ràng, không circular
  [ ] Interface đơn giản, đủ flexible
  [ ] Có tests hoặc examples
```

---

## 💡 Lời Khuyên Từ Thực Tế

```
1. Flat trước, refactor sau:
   Bắt đầu với Flat Pattern. Chỉ move sang Nested khi
   thực sự cần. Đừng over-engineer từ đầu.

2. Wrapper cho public modules:
   Mỗi khi dùng public module trong team lớn,
   wrap nó để enforce company conventions.

3. "Module" ≠ "Reusable":
   Không phải mọi thứ đều cần thành module có thể tái sử dụng.
   Root module cũng là module — không cần abstract hoá quá mức.

4. Terraform graph là bạn:
   terraform graph | dot -Tsvg > graph.svg
   Visualize dependencies để phát hiện circular deps sớm.

5. Plan output không nói dối:
   Trước khi apply bất kỳ thay đổi module nào,
   đọc kỹ plan output — tìm "destroy" keywords.
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Kết Thúc:** Phần 03-modules hoàn thành ✅
**Tiếp Theo:** [04-workspaces-environments](../04-workspaces-environments/README.md)
