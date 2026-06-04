# Providers & Resources — Nhà Cung Cấp & Tài Nguyên

> Provider là cầu nối giữa Terraform và cloud API. Resource là đơn vị hạ tầng mà Terraform quản lý. Hiểu rõ hai khái niệm này là hiểu cách Terraform thực sự hoạt động.

---

## 📚 Mục Lục

1. [Provider Là Gì](#1-provider-là-gì)
2. [Cấu Hình Provider](#2-cấu-hình-provider)
3. [Provider Authentication — Xác Thực](#3-provider-authentication---xác-thực)
4. [Multiple Providers — Nhiều Provider](#4-multiple-providers---nhiều-provider)
5. [Resource Là Gì](#5-resource-là-gì)
6. [Resource Lifecycle — Vòng Đời Tài Nguyên](#6-resource-lifecycle---vòng-đời-tài-nguyên)
7. [Resource Dependencies — Phụ Thuộc Tài Nguyên](#7-resource-dependencies---phụ-thuộc-tài-nguyên)
8. [Data Sources — Nguồn Dữ Liệu](#8-data-sources---nguồn-dữ-liệu)
9. [Meta-arguments — Tham Số Đặc Biệt](#9-meta-arguments---tham-số-đặc-biệt)
10. [Practical Example — Ví Dụ Thực Tế](#10-practical-example---ví-dụ-thực-tế)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)

---

## 1. Provider Là Gì

**Provider — Nhà cung cấp** là plugin — tiện ích mở rộng — mà Terraform dùng để tương tác với cloud platform, SaaS service, hoặc bất kỳ API nào.

```
Code HCL của bạn
      │
      ▼
Terraform Core (terraform plan, apply)
      │
      ▼  (gọi plugin)
Provider Plugin (aws, google, azurerm...)
      │
      ▼  (gọi API)
Cloud API (AWS API, GCP API, Azure API...)
      │
      ▼
Tài nguyên được tạo/cập nhật/xóa
```

### Các Provider Phổ Biến

| Provider | Source | Mô Tả |
|----------|--------|-------|
| `hashicorp/aws` | Terraform Registry | Amazon Web Services |
| `hashicorp/google` | Terraform Registry | Google Cloud Platform |
| `hashicorp/azurerm` | Terraform Registry | Microsoft Azure |
| `hashicorp/kubernetes` | Terraform Registry | Kubernetes cluster |
| `hashicorp/helm` | Terraform Registry | Helm charts — Kubernetes package manager |
| `hashicorp/random` | Terraform Registry | Tạo giá trị ngẫu nhiên |
| `hashicorp/local` | Terraform Registry | Tài nguyên local file system |
| `hashicorp/null` | Terraform Registry | Null resource cho provisioners |
| `datadog/datadog` | Terraform Registry | Datadog monitoring — Giám sát |
| `pagerduty/pagerduty` | Terraform Registry | PagerDuty on-call |
| `github/github` | Terraform Registry | Quản lý GitHub repos, teams |

### Provider Versioning — Quản Lý Phiên Bản Provider

```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # registry.terraform.io/hashicorp/aws
      version = "~> 5.31"         # >= 5.31, < 6.0
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

Sau khi `terraform init`, file `.terraform.lock.hcl` — lock file — được tạo ra, pin chính xác version đang dùng. Commit file này vào git để đảm bảo team dùng cùng provider version.

---

## 2. Cấu Hình Provider

### Cấu Hình Cơ Bản

```hcl
# providers.tf

# AWS Provider
provider "aws" {
  region = "ap-southeast-1"  # Singapore
}

# GCP Provider
provider "google" {
  project = "my-project-id"
  region  = "asia-southeast1"
  zone    = "asia-southeast1-a"
}

# Azure Provider
provider "azurerm" {
  features {}  # Block này bắt buộc, có thể để trống
  subscription_id = var.azure_subscription_id
}
```

### Dùng Variables Trong Provider

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project
      Environment = var.environment
    }
  }
}
```

`default_tags` — Nhãn mặc định — tự động áp dụng cho tất cả resources trong provider đó, tránh phải lặp lại `tags` ở mỗi resource.

---

## 3. Provider Authentication — Xác Thực

### AWS Authentication — Xác Thực AWS

```hcl
# ❌ KHÔNG BAO GIỜ hardcode credentials trong code
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"   # NGUY HIỂM!
  secret_key = "wJalrXUtnFEMI/K7MDENG" # NGUY HIỂM!
}
```

```hcl
# ✅ Cách đúng 1: Environment variables — Biến môi trường
# export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
# export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG"
# export AWS_DEFAULT_REGION="ap-southeast-1"
provider "aws" {
  region = var.aws_region
  # Tự động lấy từ environment variables
}

# ✅ Cách đúng 2: AWS Profile — Hồ sơ AWS (~/.aws/credentials)
provider "aws" {
  region  = var.aws_region
  profile = "my-project-profile"
}

# ✅ Cách đúng 3: IAM Role (tốt nhất cho CI/CD)
# EC2 instance profile, ECS task role, GitHub Actions OIDC...
provider "aws" {
  region = var.aws_region
  # Tự động assume role từ instance/task metadata
}

# ✅ Cách đúng 4: Assume Role — Đảm nhận vai trò (cho cross-account)
provider "aws" {
  region = var.aws_region
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
  }
}
```

### GCP Authentication — Xác Thực GCP

```bash
# Cách 1: Application Default Credentials — ADC
gcloud auth application-default login

# Cách 2: Service Account Key (tránh dùng trong production)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"

# Cách 3: Workload Identity Federation — Tốt nhất cho CI/CD
```

---

## 4. Multiple Providers — Nhiều Provider

### Multi-region — Đa Vùng (AWS)

```hcl
# Provider chính — us-east-1
provider "aws" {
  region = "us-east-1"
  alias  = "us_east_1"
}

# Provider phụ — ap-southeast-1
provider "aws" {
  region = "ap-southeast-1"
  alias  = "singapore"
}

# Dùng provider cụ thể bằng provider argument
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east_1
  bucket   = "myapp-us-east-1"
}

resource "aws_s3_bucket" "sg_bucket" {
  provider = aws.singapore
  bucket   = "myapp-ap-southeast-1"
}
```

### Multi-account — Đa Tài Khoản

```hcl
# Tài khoản networking (account A)
provider "aws" {
  region = "ap-southeast-1"
  alias  = "networking"
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

# Tài khoản application (account B)
provider "aws" {
  region = "ap-southeast-1"
  alias  = "application"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}
```

### Multi-cloud — Đa Cloud

```hcl
provider "aws" {
  region = "ap-southeast-1"
}

provider "google" {
  project = "my-gcp-project"
  region  = "asia-southeast1"
}

# AWS resource
resource "aws_s3_bucket" "backup" {
  bucket = "myapp-backups"
}

# GCP resource
resource "google_storage_bucket" "assets" {
  name     = "myapp-assets"
  location = "ASIA"
}
```

---

## 5. Resource Là Gì

**Resource — Tài nguyên** là component — thành phần — hạ tầng mà Terraform tạo, cập nhật, và xóa. Mỗi resource đại diện cho một đối tượng trong cloud (EC2 instance, S3 bucket, RDS database...).

### Anatomy of a Resource — Cấu Tạo Resource

```hcl
resource "<PROVIDER>_<TYPE>" "<LOCAL_NAME>" {
  # Arguments — Tham số cấu hình resource
  argument_1 = value_1
  argument_2 = value_2

  # Nested blocks — Khối lồng nhau
  nested_block {
    sub_argument = value
  }

  # Meta-arguments — Tham số đặc biệt của Terraform
  depends_on = [...]
  lifecycle { ... }
  count = N
  for_each = { ... }
  provider = provider.alias
}
```

### Ví Dụ Thực Tế

```hcl
resource "aws_instance" "web" {
  # Required arguments — Tham số bắt buộc
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  # Optional arguments — Tham số tùy chọn
  key_name               = aws_key_pair.deployer.key_name
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = module.vpc.public_subnets[0]

  # Nested block
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    delete_on_termination = true
    encrypted             = true
  }

  user_data = templatefile("${path.module}/templates/user_data.sh.tpl", {
    db_host = aws_db_instance.main.address
  })

  tags = {
    Name = "web-${var.environment}"
  }
}
```

### Resource Address — Địa Chỉ Resource

```
aws_instance.web            → Resource đơn
aws_instance.web[0]         → Resource với count, index 0
aws_instance.web["prod"]    → Resource với for_each, key "prod"
module.vpc.aws_vpc.main     → Resource trong module
```

---

## 6. Resource Lifecycle — Vòng Đời Tài Nguyên

### Terraform Quyết Định Hành Động Như Thế Nào

```
Terraform plan:
  1. Đọc desired state — Trạng thái mong muốn (từ .tf files)
  2. Đọc current state — Trạng thái hiện tại (từ state file)
  3. Gọi provider để refresh actual state — Trạng thái thực tế
  4. So sánh: desired vs actual
  5. Tạo execution plan — Kế hoạch thực thi

Kết quả:
  + create  → Resource chưa tồn tại, sẽ tạo mới
  ~ update  → Resource tồn tại, một số attributes thay đổi
  - destroy → Resource không còn trong config, sẽ xóa
  -/+ replace → Attribute không thể update in-place, phải recreate
```

### Lifecycle Meta-argument

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    # Tạo mới trước khi xóa cũ (zero downtime replacement — thay thế không downtime)
    create_before_destroy = true

    # Ngăn Terraform xóa resource này
    prevent_destroy = true

    # Bỏ qua thay đổi của một số attributes
    # Hữu ích khi attribute bị thay đổi bên ngoài Terraform
    ignore_changes = [
      tags["LastUpdated"],
      user_data,           # Không recreate khi user_data thay đổi
    ]

    # Custom condition — Điều kiện tùy chỉnh (Terraform >= 1.2)
    precondition {
      condition     = data.aws_ami.ubuntu.architecture == "x86_64"
      error_message = "AMI phải là x86_64 architecture."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance phải có public IP."
    }
  }
}
```

### Tainted Resources — Tài Nguyên Bị Đánh Dấu

```bash
# Đánh dấu resource cần recreate dù config không thay đổi
terraform taint aws_instance.web

# Gỡ đánh dấu
terraform untaint aws_instance.web

# Terraform >= 1.0: dùng -replace thay cho taint
terraform apply -replace=aws_instance.web
```

---

## 7. Resource Dependencies — Phụ Thuộc Tài Nguyên

### Implicit Dependencies — Phụ Thuộc Ngầm (Ưu Tiên Dùng)

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Ngầm phụ thuộc vào aws_vpc.main
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # Ngầm phụ thuộc vào aws_subnet.public
  # ...
}

# Terraform tự build dependency graph — Đồ thị phụ thuộc:
# aws_vpc.main → aws_subnet.public → aws_instance.web
# Tạo theo thứ tự này; xóa theo thứ tự ngược lại
```

### Explicit Dependencies — Phụ Thuộc Tường Minh

```hcl
# Dùng depends_on khi phụ thuộc không thể suy ra từ attribute references
resource "aws_iam_role_policy" "terraform_policy" {
  role   = aws_iam_role.terraform_role.id
  policy = data.aws_iam_policy_document.terraform.json
}

resource "aws_s3_bucket" "state" {
  bucket = "myapp-terraform-state"

  # IAM policy phải sẵn sàng trước khi tạo bucket
  # (không có attribute reference, nên phải explicit)
  depends_on = [aws_iam_role_policy.terraform_policy]
}
```

### Dependency Graph — Đồ Thị Phụ Thuộc

```bash
# Xem dependency graph dưới dạng DOT format
terraform graph | dot -Tpng > graph.png

# Xem text format
terraform graph
```

---

## 8. Data Sources — Nguồn Dữ Liệu

Data source — Nguồn dữ liệu — cho phép **đọc** thông tin từ provider mà **không tạo** resource mới. Dùng để tham chiếu tài nguyên đã tồn tại bên ngoài Terraform.

```hcl
# Tìm AMI — Amazon Machine Image — mới nhất của Ubuntu
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Lấy thông tin VPC đã tồn tại (tạo bằng tay hoặc Terraform khác)
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Lấy thông tin tài khoản AWS hiện tại
data "aws_caller_identity" "current" {}

# Lấy các available zones — Vùng khả dụng
data "aws_availability_zones" "available" {
  state = "available"
}

# SSM Parameter Store — Đọc secret từ AWS Systems Manager
data "aws_ssm_parameter" "db_password" {
  name = "/myapp/${var.environment}/db_password"
}

# Dùng trong resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id      # Dùng AMI từ data source
  instance_type = "t3.micro"
  subnet_id     = data.aws_vpc.existing.id    # Dùng VPC từ data source

  # data.aws_caller_identity.current.account_id → "123456789012"
  # data.aws_availability_zones.available.names → ["ap-southeast-1a", ...]
}
```

### Data Source vs Resource

| | Resource | Data Source |
|--|---------|-------------|
| **Mục đích** | Tạo/Quản lý resource | Đọc thông tin resource |
| **Hành động** | Create, Update, Delete | Read Only |
| **Trong state** | Có | Có (cached) |
| **Prefix** | `resource "type" "name"` | `data "type" "name"` |
| **Tham chiếu** | `type.name.attr` | `data.type.name.attr` |

---

## 9. Meta-arguments — Tham Số Đặc Biệt

### depends_on

```hcl
resource "aws_s3_bucket_policy" "logs" {
  bucket = aws_s3_bucket.logs.id
  policy = data.aws_iam_policy_document.logs.json

  # Chỉ apply policy sau khi bucket đã tạo xong hoàn toàn
  depends_on = [aws_s3_bucket.logs]
}
```

### count

```hcl
resource "aws_instance" "web" {
  count         = var.instance_count  # 0, 1, hoặc N instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}
```

### for_each

```hcl
resource "aws_iam_user" "developers" {
  for_each = toset(var.developer_names)
  name     = each.value
}
```

### provider

```hcl
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east_1  # Dùng provider alias
  bucket   = "my-us-bucket"
}
```

### lifecycle

Đã trình bày ở [mục 6](#6-resource-lifecycle---vòng-đời-tài-nguyên).

---

## 10. Practical Example — Ví Dụ Thực Tế

### Tạo EC2 Instance Hoàn Chỉnh

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project
      Environment = var.environment
    }
  }
}

# data.tf — Data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

data "aws_vpc" "default" {
  default = true
}

# main.tf — Resources
resource "aws_security_group" "web" {
  name        = "${var.project}-${var.environment}-web-sg"
  description = "Security group cho web servers"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP traffic"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS traffic"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOT
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
  EOT

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "${var.project}-${var.environment}-web"
  }
}

# outputs.tf
output "instance_public_ip" {
  description = "IP công khai của web server"
  value       = aws_instance.web.public_ip
}

output "instance_id" {
  description = "ID của EC2 instance"
  value       = aws_instance.web.id
}
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: Provider là gì trong Terraform? Nó hoạt động như thế nào?**

> Provider là plugin — tiện ích mở rộng — mà Terraform dùng để giao tiếp với cloud API. Khi chạy `terraform init`, Terraform tải provider plugin về máy. Khi `apply`, Terraform Core gọi provider plugin, plugin dịch resource declarations — khai báo tài nguyên — thành API calls — lời gọi API — đến cloud provider. Có hơn 3000 providers trên Terraform Registry, từ AWS, GCP, Azure đến GitHub, Datadog, PagerDuty.

**Q: Sự khác biệt giữa resource và data source?**

> Resource quản lý lifecycle — vòng đời — của hạ tầng: create, update, delete. Data source chỉ đọc — read-only — thông tin từ provider, không tạo hay thay đổi gì. Ví dụ: dùng data source để tìm AMI mới nhất của Ubuntu, dùng resource để tạo EC2 instance với AMI đó.

**Q: Implicit vs explicit dependency — phụ thuộc ngầm vs tường minh?**

> Implicit — ngầm: Terraform tự detect khi bạn tham chiếu attribute của resource khác (e.g., `subnet_id = aws_subnet.public.id`). Terraform build dependency graph — đồ thị phụ thuộc — từ các tham chiếu này. Explicit — tường minh: Dùng `depends_on` khi có phụ thuộc nhưng không có attribute reference trực tiếp, ví dụ khi resource phụ thuộc vào side effect — tác dụng phụ — của resource khác.

**Q: `lifecycle { prevent_destroy = true }` dùng trong trường hợp nào?**

> Dùng cho production resources quan trọng như RDS production database, S3 state bucket. Khi có `prevent_destroy = true`, `terraform destroy` hoặc bất kỳ plan nào dẫn đến xóa resource sẽ fail với error. Đây là safeguard — biện pháp bảo vệ — chống xóa nhầm.

---

## 📋 Checklist

- [ ] Giải thích được role của provider trong Terraform
- [ ] Cấu hình authentication cho AWS/GCP không hardcode credentials
- [ ] Viết resource với lifecycle meta-argument
- [ ] Phân biệt data source vs resource với ví dụ cụ thể
- [ ] Hiểu implicit vs explicit dependency và khi nào dùng depends_on

---

## 🔗 Liên Kết

- ← [2. HCL Syntax](./2-hcl-syntax.md)
- → [4. Variables & Outputs](./4-variables-outputs.md)

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
