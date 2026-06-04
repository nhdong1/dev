# HCL — HashiCorp Configuration Language — Cú Pháp Đầy Đủ

> HCL là ngôn ngữ cấu hình được thiết kế để vừa dễ đọc với người, vừa dễ parse — phân tích cú pháp — với máy. Nắm vững HCL là bước đầu tiên để viết Terraform chuyên nghiệp.

---

## 📚 Mục Lục

1. [Cấu Trúc File Terraform](#1-cấu-trúc-file-terraform)
2. [Kiểu Dữ Liệu Cơ Bản](#2-kiểu-dữ-liệu-cơ-bản)
3. [Blocks — Khối Cấu Hình](#3-blocks---khối-cấu-hình)
4. [Expressions — Biểu Thức](#4-expressions---biểu-thức)
5. [Functions — Hàm Tích Hợp](#5-functions---hàm-tích-hợp)
6. [Conditionals — Điều Kiện](#6-conditionals---điều-kiện)
7. [Loops — Vòng Lặp](#7-loops---vòng-lặp)
8. [String Templates — Mẫu Chuỗi](#8-string-templates---mẫu-chuỗi)
9. [Comments — Ghi Chú](#9-comments---ghi-chú)
10. [Cấu Trúc File Theo Quy Ước](#10-cấu-trúc-file-theo-quy-ước)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)

---

## 1. Cấu Trúc File Terraform

### Tổ Chức File Chuẩn

```
module-or-project/
├── main.tf           # Resources chính
├── variables.tf      # Khai báo tất cả variables
├── outputs.tf        # Khai báo tất cả outputs
├── providers.tf      # Cấu hình provider
├── versions.tf       # Terraform version constraints
├── locals.tf         # Local values (nếu nhiều)
└── terraform.tfvars  # Values cho variables (KHÔNG commit lên git nếu có secret)
```

### File `versions.tf` — Khai Báo Phiên Bản

```hcl
terraform {
  # Yêu cầu phiên bản Terraform tối thiểu
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # Tương đương >= 5.0, < 6.0
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.4"
    }
  }
}
```

### Version Constraints — Ràng Buộc Phiên Bản

| Syntax | Ý Nghĩa | Ví Dụ |
|--------|---------|-------|
| `= 1.2.3` | Đúng phiên bản này | `= 5.31.0` |
| `!= 1.2.3` | Trừ phiên bản này | `!= 5.0.0` |
| `> 1.2.3` | Lớn hơn | `> 5.0` |
| `>= 1.2.3` | Lớn hơn hoặc bằng | `>= 5.0` |
| `< 1.2.3` | Nhỏ hơn | `< 6.0` |
| `~> 1.2` | Pessimistic constraint — >= 1.2, < 2.0 | `~> 5.0` |
| `~> 1.2.3` | >= 1.2.3, < 1.3.0 | `~> 5.31.0` |

**Khuyến nghị:** Dùng `~> major.minor` để cho phép patch updates nhưng tránh breaking changes.

---

## 2. Kiểu Dữ Liệu Cơ Bản

### Primitive Types — Kiểu Nguyên Thủy

```hcl
# string — Chuỗi ký tự
variable "region" {
  type    = string
  default = "ap-southeast-1"
}

# number — Số (integer và float)
variable "instance_count" {
  type    = number
  default = 3
}

variable "cpu_threshold" {
  type    = number
  default = 75.5
}

# bool — Boolean — Giá trị đúng/sai
variable "enable_monitoring" {
  type    = bool
  default = true
}
```

### Collection Types — Kiểu Tập Hợp

```hcl
# list — Danh sách có thứ tự (có thể trùng lặp)
variable "availability_zones" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

# set — Tập hợp không có thứ tự (không trùng lặp)
variable "allowed_ports" {
  type    = set(number)
  default = [80, 443, 8080]
}

# map — Bản đồ key-value
variable "tags" {
  type = map(string)
  default = {
    Project     = "myapp"
    ManagedBy   = "terraform"
    Environment = "dev"
  }
}

# Truy cập giá trị
# var.tags["Project"]           → "myapp"
# var.availability_zones[0]     → "ap-southeast-1a"
```

### Structural Types — Kiểu Có Cấu Trúc

```hcl
# object — Đối tượng với schema cố định
variable "database_config" {
  type = object({
    engine         = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
  }
}

# tuple — Bộ với các kiểu khác nhau (ít dùng)
variable "server_config" {
  type    = tuple([string, number, bool])
  default = ["t3.micro", 1, true]
}
```

### any — Kiểu Bất Kỳ (Dùng Cẩn Thận)

```hcl
# any — bỏ qua type checking — kiểm tra kiểu
# Chỉ dùng khi cần linh hoạt, không biết trước kiểu
variable "extra_config" {
  type    = any
  default = {}
}
```

---

## 3. Blocks — Khối Cấu Hình

HCL tổ chức cấu hình thành các **blocks — khối**. Mỗi block có keyword — từ khóa, optional labels — nhãn tùy chọn, và body — phần thân.

```
<BLOCK_TYPE> "<BLOCK_LABEL_1>" "<BLOCK_LABEL_2>" {
  # Block body — phần thân
  <ATTRIBUTE> = <VALUE>

  <NESTED_BLOCK> {
    ...
  }
}
```

### Block Types Quan Trọng

```hcl
# terraform block — Cấu hình Terraform core
terraform {
  required_version = ">= 1.6"
  required_providers { ... }
  backend "s3" { ... }
}

# provider block — Cấu hình provider
provider "aws" {
  region  = "ap-southeast-1"
  profile = "default"
}

# resource block — Khai báo tài nguyên (phổ biến nhất)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# data block — Đọc dữ liệu từ provider mà không tạo resource
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical's AWS account
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# variable block — Khai báo đầu vào
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

# output block — Khai báo đầu ra
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "IP công khai của EC2 instance"
  sensitive   = false
}

# locals block — Biến cục bộ, tính toán trung gian
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  full_name = "${var.project_name}-${var.environment}"
}

# module block — Gọi module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = local.full_name
  cidr    = "10.0.0.0/16"
}
```

---

## 4. Expressions — Biểu Thức

### References — Tham Chiếu

```hcl
# Tham chiếu resource attribute
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id    # data source
  instance_type = var.instance_type          # variable
  subnet_id     = module.vpc.public_subnets[0]  # module output
}

# Tham chiếu output của resource đã tạo
resource "aws_eip" "web" {
  instance = aws_instance.web.id  # resource reference
}
```

### Cú Pháp Tham Chiếu

| Loại | Cú Pháp | Ví Dụ |
|------|---------|-------|
| Resource | `<TYPE>.<NAME>.<ATTR>` | `aws_instance.web.id` |
| Variable | `var.<NAME>` | `var.instance_type` |
| Local | `local.<NAME>` | `local.common_tags` |
| Data source | `data.<TYPE>.<NAME>.<ATTR>` | `data.aws_ami.ubuntu.id` |
| Module output | `module.<NAME>.<OUTPUT>` | `module.vpc.vpc_id` |
| Path | `path.module`, `path.root` | `"${path.module}/files"` |

### Meta-arguments — Tham Số Đặc Biệt

```hcl
resource "aws_instance" "web" {
  count = 3   # Tạo 3 instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  # Tham chiếu đến index của instance hiện tại
  tags = { Name = "web-${count.index}" }
}

# Tham chiếu: aws_instance.web[0], aws_instance.web[1], ...
```

---

## 5. Functions — Hàm Tích Hợp

Terraform có nhiều built-in functions — hàm tích hợp. Không thể tự định nghĩa hàm (dùng locals thay thế).

### String Functions — Hàm Xử Lý Chuỗi

```hcl
locals {
  # format — Định dạng chuỗi
  bucket_name = format("myapp-%s-%s", var.environment, var.region)
  # → "myapp-prod-ap-southeast-1"

  # upper, lower — Chuyển hoa/thường
  env_upper = upper(var.environment)   # "PROD"
  env_lower = lower(var.environment)   # "prod"

  # replace — Thay thế chuỗi con
  safe_name = replace(var.app_name, "_", "-")  # "my_app" → "my-app"

  # split — Tách chuỗi thành list
  parts = split(",", "a,b,c")   # ["a", "b", "c"]

  # join — Nối list thành chuỗi
  joined = join(", ", ["a", "b", "c"])  # "a, b, c"

  # trimspace — Xóa khoảng trắng đầu/cuối
  clean = trimspace("  hello  ")  # "hello"

  # substr — Lấy chuỗi con
  short = substr("hello-world", 0, 5)  # "hello"

  # length — Độ dài
  len = length("hello")  # 5
}
```

### Collection Functions — Hàm Xử Lý Tập Hợp

```hcl
locals {
  # concat — Nối hai list
  all_subnets = concat(module.vpc.public_subnets, module.vpc.private_subnets)

  # flatten — Làm phẳng list lồng nhau
  flat = flatten([[1, 2], [3, 4]])  # [1, 2, 3, 4]

  # distinct — Loại bỏ trùng lặp
  unique_zones = distinct(["a", "b", "a", "c"])  # ["a", "b", "c"]

  # length — Số phần tử
  count = length(var.availability_zones)

  # merge — Gộp nhiều map
  all_tags = merge(
    local.common_tags,
    { "Service" = "web" }
  )

  # keys, values — Lấy keys hoặc values của map
  tag_keys = keys(var.tags)

  # lookup — Tìm giá trị trong map với default
  zone = lookup(var.region_to_zone, var.region, "unknown")

  # contains — Kiểm tra phần tử có trong list/set
  is_prod = contains(["prod", "production"], var.environment)

  # toset — Chuyển list thành set (loại trùng lặp)
  az_set = toset(var.availability_zones)

  # tolist, tomap — Chuyển đổi kiểu
  subnet_list = tolist(aws_subnet.private[*].id)
}
```

### Numeric Functions — Hàm Số Học

```hcl
locals {
  # max, min
  max_count = max(var.min_instances, 3)
  min_count = min(var.max_instances, 10)

  # floor, ceil, abs
  rounded = floor(3.7)  # 3
  ceiling = ceil(3.1)   # 4
  positive = abs(-5)    # 5
}
```

### Type Conversion — Chuyển Đổi Kiểu

```hcl
locals {
  # tostring, tonumber, tobool
  count_str = tostring(var.instance_count)  # 3 → "3"
  port_num  = tonumber("8080")              # "8080" → 8080
  flag      = tobool("true")               # "true" → true
}
```

### Encoding Functions — Hàm Mã Hóa

```hcl
locals {
  # base64encode, base64decode
  encoded = base64encode("hello")  # "aGVsbG8="

  # jsonencode, jsondecode — Quan trọng!
  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "*"
    }]
  })

  # yamlencode — Ít dùng hơn
  yaml_output = yamlencode({ key = "value" })
}
```

### File Functions — Hàm Đọc File

```hcl
locals {
  # file — Đọc nội dung file (relative path)
  user_data = file("${path.module}/scripts/init.sh")

  # templatefile — Đọc file với template substitution
  rendered_script = templatefile("${path.module}/templates/init.sh.tpl", {
    db_host = aws_db_instance.main.address
    app_port = var.app_port
  })

  # filebase64 — Đọc file và encode base64
  cert_b64 = filebase64("${path.module}/certs/cert.pem")

  # filemd5 — Hash MD5 của file (trigger resource recreation khi file thay đổi)
  script_hash = filemd5("${path.module}/scripts/init.sh")
}
```

---

## 6. Conditionals — Điều Kiện

### Ternary Expression — Biểu Thức Ba Ngôi

```hcl
# <CONDITION> ? <TRUE_VALUE> : <FALSE_VALUE>

resource "aws_db_instance" "main" {
  # Multi-AZ — Đa vùng khả dụng — chỉ bật ở production
  multi_az = var.environment == "prod" ? true : false

  # Hoặc ngắn gọn hơn
  multi_az = var.environment == "prod"
}

locals {
  # Chọn instance type theo môi trường
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  # Điều kiện phức tạp
  enable_backup = (var.environment == "prod" || var.environment == "staging") ? true : false
}
```

### Điều Kiện Với count

```hcl
# Tạo resource có điều kiện — chỉ tạo nếu enable = true
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name = "high-cpu-${var.environment}"
  # ...
}

# Tham chiếu resource có điều kiện
output "alarm_arn" {
  value = var.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
}
```

---

## 7. Loops — Vòng Lặp

### count — Đếm Số Lượng

```hcl
# Tạo 3 EC2 instances
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2
  }
}

# Tham chiếu
output "instance_ids" {
  value = aws_instance.web[*].id  # splat expression
}
```

**Hạn chế của count:** Nếu xóa phần tử ở giữa list, Terraform phải recreate — tạo lại — tất cả phần tử sau đó (index shifting — dịch chuyển index).

### for_each — Lặp Theo Map/Set (Ưu Tiên Hơn count)

```hcl
# for_each với set
resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key   # "ap-southeast-1a", etc.
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
}

# for_each với map — Linh hoạt nhất
variable "s3_buckets" {
  type = map(object({
    versioning = bool
    lifecycle  = bool
  }))
  default = {
    "assets"  = { versioning = true, lifecycle = true }
    "backups" = { versioning = true, lifecycle = false }
    "logs"    = { versioning = false, lifecycle = true }
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets

  bucket = "${var.project}-${each.key}-${var.environment}"

  tags = {
    Name    = each.key
    Purpose = each.key
  }
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.s3_buckets : k => v if v.versioning }

  bucket = aws_s3_bucket.buckets[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}

# Tham chiếu: aws_s3_bucket.buckets["assets"].id
```

### for Expressions — Biến Đổi Tập Hợp

```hcl
locals {
  # Biến đổi list thành list mới
  upper_zones = [for az in var.availability_zones : upper(az)]
  # ["AP-SOUTHEAST-1A", "AP-SOUTHEAST-1B"]

  # Lọc với điều kiện
  prod_instances = [
    for instance in var.instances : instance
    if instance.environment == "prod"
  ]

  # Biến đổi list thành map
  instance_map = {
    for instance in var.instances :
    instance.name => instance.id
  }

  # Biến đổi map thành map mới
  upper_tags = {
    for k, v in var.tags : k => upper(v)
  }

  # Phức tạp hơn: group by — nhóm theo
  instances_by_type = {
    for instance in var.instances :
    instance.type => instance...
  }
}
```

---

## 8. String Templates — Mẫu Chuỗi

### Interpolation — Nội Suy Chuỗi

```hcl
locals {
  # Cơ bản: ${expression}
  bucket_name = "myapp-${var.environment}-${var.region}"

  # Biểu thức phức tạp
  name = "web-${var.environment == "prod" ? "prod" : "non-prod"}"

  # Heredoc — Chuỗi đa dòng
  user_data = <<-EOT
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    echo "Server: ${var.environment}" > /var/www/html/index.html
    systemctl start nginx
  EOT
}
```

### Directives — Chỉ Thị Template

```hcl
locals {
  # %{if} %{else} %{endif} — Điều kiện trong template
  greeting = "Hello, %{if var.is_prod}production%{else}development%{endif} server!"

  # %{for} %{endfor} — Vòng lặp trong template
  server_list = <<-EOT
    Servers:
    %{for server in var.servers}
    - ${server.name}: ${server.ip}
    %{endfor}
  EOT
}
```

---

## 9. Comments — Ghi Chú

```hcl
# Comment một dòng — dùng phổ biến nhất

// Comment một dòng — cũng hợp lệ nhưng ít dùng

/*
  Comment
  nhiều dòng
*/

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id  # Inline comment
  instance_type = "t3.micro"
}
```

---

## 10. Cấu Trúc File Theo Quy Ước

### `main.tf` — Tài Nguyên Chính

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${local.prefix}-vpc" })
}

resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
  availability_zone = each.key
  tags = merge(local.common_tags, { Name = "${local.prefix}-public-${each.key}" })
}
```

### `variables.tf` — Khai Báo Biến

```hcl
# variables.tf
variable "project" {
  description = "Tên dự án"
  type        = string
}

variable "environment" {
  description = "Môi trường triển khai: dev, staging, prod"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment phải là dev, staging, hoặc prod."
  }
}

variable "vpc_cidr" {
  description = "CIDR block cho VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```

### `outputs.tf` — Khai Báo Đầu Ra

```hcl
# outputs.tf
output "vpc_id" {
  description = "ID của VPC đã tạo"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Danh sách ID của public subnets"
  value       = [for subnet in aws_subnet.public : subnet.id]
}
```

### `locals.tf` — Biến Cục Bộ

```hcl
# locals.tf
locals {
  prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    UpdatedAt   = formatdate("YYYY-MM-DD", timestamp())
  }
}
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: `count` vs `for_each` — khi nào dùng cái nào?**

> Dùng `for_each` hầu hết thời gian. `count` tốt khi cần tạo N bản giống nhau không có identifier — định danh riêng. `for_each` tốt hơn khi lặp qua set hoặc map vì resources được identify — định danh — bằng key thay vì index, tránh index shifting — dịch chuyển index — khi thêm/xóa phần tử giữa chừng. Ví dụ: xóa item thứ 2 trong list count-3 sẽ force replace — ép buộc tạo lại — item thứ 3; với for_each thì không.

**Q: Validation trong variable dùng để làm gì?**

> Validation block trong variable cho phép kiểm tra giá trị ngay khi `terraform plan`, thay vì đợi đến lúc `apply` mới lỗi ở API. Giúp fail fast — thất bại sớm — và cung cấp error message — thông báo lỗi — rõ ràng hơn.

**Q: Khi nào dùng `locals` thay vì `variable`?**

> `variable` — cho phép người dùng bên ngoài truyền vào, xuất hiện trong `terraform.tfvars`. `locals` — tính toán trung gian, không thể override từ bên ngoài, dùng để tránh lặp lại logic phức tạp. Nếu cần user input → variable; nếu là derived value — giá trị tính từ các giá trị khác → local.

---

## 📋 Checklist

- [ ] Phân biệt được các kiểu dữ liệu: string, number, bool, list, map, object
- [ ] Viết được version constraints cho provider
- [ ] Sử dụng được for_each với map và set
- [ ] Biết ít nhất 10 built-in functions phổ biến
- [ ] Dùng được conditional expressions và string templates

---

## 🔗 Liên Kết

- ← [1. IaC là gì](./1-what-is-iac.md)
- → [3. Providers & Resources](./3-providers-resources.md)

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
