# Terraform Workspaces — Không Gian Làm Việc

> Workspaces cho phép dùng chung một codebase nhưng quản lý nhiều state file khác nhau. Nghe hay — nhưng thực tế có nhiều giới hạn quan trọng cần biết.

---

## 📚 Mục Lục

1. [Workspace là gì?](#workspace-là-gì)
2. [Cách hoạt động](#cách-hoạt-động)
3. [Các lệnh cơ bản](#các-lệnh-cơ-bản)
4. [Dùng workspace trong code](#dùng-workspace-trong-code)
5. [Giới hạn của workspaces](#giới-hạn-của-workspaces)
6. [Khi nào nên dùng workspaces](#khi-nào-nên-dùng-workspaces)
7. [Khi nào KHÔNG nên dùng workspaces](#khi-nào-không-nên-dùng-workspaces)
8. [So sánh với directory separation](#so-sánh-với-directory-separation)
9. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Workspace là gì?

**Workspace** — Không Gian Làm Việc — là một tính năng của Terraform cho phép một thư mục cấu hình (configuration directory) quản lý **nhiều state file độc lập**.

Mỗi workspace có:
- **State file riêng** — không chia sẻ với workspace khác
- **Cùng một codebase** — dùng chung tất cả `.tf` files
- **Cùng một backend** — nhưng lưu ở sub-path riêng

```
Mặc định, mọi dự án Terraform đều ở workspace "default"
```

### Ví dụ Trực Quan

Không có workspace (tất cả dùng chung):
```
terraform.tfstate  ← Tất cả môi trường dùng chung — nguy hiểm!
```

Với workspace:
```
terraform.tfstate.d/
├── dev/
│   └── terraform.tfstate     ← State riêng cho dev
├── staging/
│   └── terraform.tfstate     ← State riêng cho staging
└── prod/
    └── terraform.tfstate     ← State riêng cho prod
```

---

## Cách hoạt động

Khi bạn tạo workspace mới và switch sang:
1. Terraform tạo một **state file riêng** cho workspace đó
2. Mọi `terraform plan` / `apply` chỉ ảnh hưởng đến state của workspace hiện tại
3. Tài nguyên được tạo ra là **độc lập hoàn toàn** với workspace khác

### Backend Storage — Nơi Lưu State Theo Workspace

**Local backend:**
```
terraform.tfstate.d/
├── dev/terraform.tfstate
└── prod/terraform.tfstate
```

**S3 backend:**
```
s3://my-bucket/
├── env:/dev/terraform.tfstate
└── env:/prod/terraform.tfstate
```

**Terraform Cloud:**
```
Workspace "myapp-dev"   → State riêng
Workspace "myapp-prod"  → State riêng
```

---

## Các lệnh cơ bản

```bash
# Xem tất cả workspace hiện có
terraform workspace list

# Output:
# * default        ← Dấu * là workspace đang dùng
#   dev
#   staging
#   prod

# Tạo workspace mới
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Chuyển sang workspace khác
terraform workspace select prod

# Xem workspace đang dùng
terraform workspace show
# Output: prod

# Xóa workspace (phải ở workspace khác, không được xóa workspace đang dùng)
terraform workspace select default
terraform workspace delete dev   # Cẩn thận! Xóa state luôn!
```

> **Cảnh báo:** `terraform workspace delete` xóa cả state file. Nếu còn tài nguyên đang quản lý, Terraform sẽ mất track — mất theo dõi — chúng.

---

## Dùng workspace trong code

### Truy cập tên workspace hiện tại

```hcl
# Biến built-in — có sẵn — của Terraform
terraform.workspace  # Trả về tên workspace hiện tại: "dev", "prod", ...
```

### Ví dụ: Phân biệt cấu hình theo workspace

```hcl
# variables.tf
locals {
  # Map cấu hình theo workspace
  env_config = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
      db_size       = "db.t3.small"
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 1
      max_size      = 4
      db_size       = "db.t3.medium"
    }
    prod = {
      instance_type = "t3.medium"
      min_size      = 3
      max_size      = 10
      db_size       = "db.r5.large"
    }
  }

  # Lấy config của workspace hiện tại, fallback về dev nếu không tìm thấy
  config = lookup(local.env_config, terraform.workspace, local.env_config["dev"])
}

# Dùng config trong resource
resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  
  tags = {
    Environment = terraform.workspace
    Name        = "app-${terraform.workspace}"
  }
}

resource "aws_autoscaling_group" "app" {
  min_size = local.config.min_size
  max_size = local.config.max_size
  # ...
}
```

### Ví dụ: Naming convention — Quy tắc đặt tên — theo workspace

```hcl
# Tự động thêm tên môi trường vào tất cả tài nguyên
resource "aws_s3_bucket" "data" {
  bucket = "myapp-${terraform.workspace}-data-${random_id.suffix.hex}"
  
  tags = {
    Environment = terraform.workspace
    ManagedBy   = "terraform"
  }
}

resource "aws_db_instance" "main" {
  identifier = "myapp-${terraform.workspace}-db"
  # ...
}
```

### Ví dụ: Điều kiện dựa theo workspace

```hcl
# Chỉ tạo một số tài nguyên ở production
resource "aws_cloudwatch_alarm" "high_cpu" {
  # Chỉ tạo alarm ở staging và prod
  count = terraform.workspace != "dev" ? 1 : 0
  
  alarm_name = "high-cpu-${terraform.workspace}"
  # ...
}

# Dùng backend S3 với path riêng theo workspace
# (Terraform tự xử lý, không cần config thêm)
```

---

## Giới hạn của workspaces

### Giới hạn 1: Cùng backend — Không cách ly hoàn toàn

Tất cả workspace dùng **cùng một backend** (ví dụ cùng S3 bucket). Điều này có nghĩa:
- Cùng AWS account / GCP project
- Cùng permissions — Quyền truy cập
- Người có quyền workspace dev cũng có thể truy cập workspace prod

```
Rủi ro: Developer có thể vô tình chạy lệnh ở workspace prod!
```

### Giới hạn 2: Khó phát hiện workspace hiện tại

Không có gì trong terminal nhắc bạn đang ở workspace nào, trừ khi chạy `terraform workspace show`. Tai nạn phổ biến:

```bash
# Developer nghĩ mình đang ở workspace dev...
terraform workspace show
# dev   ← OK

# Sau đó làm việc khác, quên switch workspace
# ...nhiều giờ sau...
terraform apply   # Ai biết đang ở workspace nào?!
```

### Giới hạn 3: Code phải xử lý tất cả workspace

Khi dùng `terraform.workspace` để phân nhánh logic, code trở nên phức tạp và dễ có bug:

```hcl
# Dần dần code này sẽ trở thành "mã spaghetti" — code rối rắm
locals {
  config = terraform.workspace == "prod" ? var.prod_config : (
    terraform.workspace == "staging" ? var.staging_config : var.dev_config
  )
}
```

### Giới hạn 4: Không hỗ trợ variable file tự động

Workspace không tự động load `.tfvars` file theo tên workspace. Bạn phải truyền thủ công:

```bash
# Phải nhớ truyền đúng file
terraform apply -var-file="envs/prod.tfvars"

# Nếu quên, Terraform dùng giá trị default — có thể gây sai
```

### Giới hạn 5: Khó quản lý ở scale lớn

Với 10+ môi trường (dev, staging, prod, qa, uat, demo, ...), workspace không còn gọn gàng nữa.

---

## Khi nào nên dùng workspaces

Workspaces **phù hợp** trong những tình huống sau:

### 1. Hạ tầng tạm thời theo feature branch

```bash
# Mỗi feature branch có môi trường test riêng
git checkout feature/new-api
terraform workspace new feature-new-api
terraform apply   # Tạo môi trường test riêng

# Sau khi merge PR — Pull Request
terraform workspace select feature-new-api
terraform destroy  # Dọn dẹp
terraform workspace delete feature-new-api
```

### 2. Hạ tầng gần như giống nhau

Khi dev/staging/prod chỉ khác nhau về kích thước (instance type, số lượng), không khác về cấu trúc.

### 3. Dự án nhỏ, team nhỏ

Khi team hiểu rõ cần check workspace trước khi apply, và rủi ro được chấp nhận.

---

## Khi nào KHÔNG nên dùng workspaces

### ❌ Không dùng khi cần cách ly hoàn toàn

Production cần ở **AWS account riêng**, không chỉ workspace riêng.

### ❌ Không dùng khi cấu hình khác nhau nhiều

Nếu prod có VPN, WAF — Web Application Firewall, Multi-AZ RDS mà dev không có, directory separation — phân tách thư mục — rõ ràng hơn.

### ❌ Không dùng trong tổ chức lớn

Khi nhiều team cùng làm việc, khả năng lẫn workspace rất cao.

### ❌ Terraform Cloud Enterprise

Terraform Cloud có "Workspaces" riêng (khái niệm khác hoàn toàn). Đừng nhầm lẫn.

---

## So sánh với directory separation

### Terraform Workspaces

```
project/
├── main.tf
├── variables.tf
└── outputs.tf

# Chạy lệnh:
terraform workspace select prod
terraform apply -var-file="prod.tfvars"
```

**Ưu điểm:**
- Ít file hơn, gọn hơn
- Dễ bắt đầu

**Nhược điểm:**
- Dễ nhầm workspace
- Cùng backend — không cách ly
- Code phức tạp khi nhiều điều kiện

---

### Directory Separation — Phân Tách Thư Mục

```
project/
├── modules/
│   └── app/
└── environments/
    ├── dev/
    │   ├── main.tf      ← Gọi module app
    │   ├── backend.tf   ← Backend riêng cho dev
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── backend.tf   ← Backend riêng cho staging
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf   ← Backend riêng cho prod
        └── terraform.tfvars
```

**Ưu điểm:**
- Rõ ràng, không thể nhầm — bạn `cd` vào đúng thư mục
- Backend riêng biệt hoàn toàn
- Không cần logic điều kiện trong code

**Nhược điểm:**
- Lặp code nhiều (DRY violation — vi phạm nguyên tắc Đừng Lặp Lại)
- Cần Terragrunt hoặc kỷ luật cao để giữ nhất quán

---

## Câu hỏi phỏng vấn

### Q1: Terraform Workspaces là gì? Có những giới hạn gì?

**Trả lời:**

Terraform Workspaces là tính năng cho phép một codebase quản lý nhiều state file độc lập. Mỗi workspace có state riêng nhưng dùng chung backend và cùng một codebase.

**Giới hạn chính:**
1. Cùng backend — không cách ly hoàn toàn về permissions
2. Dễ nhầm workspace đang dùng (không có indicator rõ ràng)
3. Code phức tạp khi phải xử lý logic khác nhau giữa workspace
4. Không tự động load `.tfvars` file theo workspace
5. Không phù hợp khi cần multi-account isolation — Cách ly đa tài khoản

---

### Q2: Khi nào nên dùng workspaces, khi nào nên dùng directory separation?

**Trả lời:**

| Tình Huống | Chọn |
|------------|------|
| Hạ tầng gần như giống nhau, chỉ khác kích thước | Workspaces |
| Tạo môi trường tạm thời cho feature branch | Workspaces |
| Cần cách ly hoàn toàn (AWS account riêng) | Directory Separation |
| Cấu hình khác nhau nhiều giữa môi trường | Directory Separation |
| Tổ chức lớn, nhiều team | Directory Separation + Terragrunt |

---

### Q3: Làm sao để an toàn hơn khi dùng workspaces?

**Trả lời:**

```bash
# 1. Luôn kiểm tra workspace trước khi apply
terraform workspace show

# 2. Thêm confirmation cho production
# Trong CI/CD, yêu cầu manual approval khi workspace = "prod"

# 3. Dùng prompt shell để hiển thị workspace hiện tại
# Thêm vào .bashrc hoặc .zshrc:
export TF_WORKSPACE_DISPLAY=$(terraform workspace show 2>/dev/null)
```

```hcl
# 4. Thêm guard — Bảo vệ — trong code
variable "expected_workspace" {
  description = "Workspace phải khớp để tránh apply nhầm"
  type        = string
}

resource "null_resource" "workspace_check" {
  lifecycle {
    precondition {
      condition     = terraform.workspace == var.expected_workspace
      error_message = "Workspace hiện tại (${terraform.workspace}) không khớp với expected (${var.expected_workspace})!"
    }
  }
}
```

---

### Q4: Terraform Workspace trong Terraform Cloud có khác không?

**Trả lời:**

Khác hoàn toàn. Trong Terraform Cloud — Đám Mây Terraform:

- Mỗi "Workspace" là một **dự án độc lập** với repository, variables, và run history — Lịch sử chạy — riêng
- Khái niệm giống với "Project" hơn là workspace trong Terraform CLI
- Khi dùng Terraform Cloud, thường **không dùng** `terraform workspace` commands

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **Luôn check workspace trước apply:** `terraform workspace show`
2. **Đặt tên workspace rõ ràng:** `dev`, `staging`, `prod` (không dùng `1`, `2`, `3`)
3. **Dùng workspace chỉ cho môi trường tương tự nhau** — không phù hợp cho cấu hình khác nhau nhiều
4. **Cân nhắc directory separation** cho production workload quan trọng
5. **Không xóa workspace** khi còn tài nguyên đang được quản lý

---

**Tiếp theo:** [2-environment-separation.md](./2-environment-separation.md) — Chiến lược phân tách môi trường
