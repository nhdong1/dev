# Vòng Đời Terraform: init → plan → apply → destroy

> Terraform có 4 lệnh cốt lõi tạo thành một workflow — quy trình — hoàn chỉnh. Hiểu chính xác điều gì xảy ra bên trong mỗi lệnh là chìa khóa để debug — gỡ lỗi — hiệu quả và tránh sự cố production.

---

## 📚 Mục Lục

1. [Tổng Quan Vòng Đời](#1-tổng-quan-vòng-đời)
2. [terraform init — Khởi Tạo](#2-terraform-init---khởi-tạo)
3. [terraform plan — Lập Kế Hoạch](#3-terraform-plan---lập-kế-hoạch)
4. [terraform apply — Áp Dụng](#4-terraform-apply---áp-dụng)
5. [terraform destroy — Hủy](#5-terraform-destroy---hủy)
6. [Các Lệnh Hỗ Trợ Quan Trọng](#6-các-lệnh-hỗ-trợ-quan-trọng)
7. [Terraform Workflow Trong Team](#7-terraform-workflow-trong-team)
8. [Xử Lý Lỗi Thường Gặp](#8-xử-lý-lỗi-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Vòng Đời

```
┌─────────────────────────────────────────────────────────┐
│              Terraform Workflow — Quy Trình              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ① terraform init                                       │
│     ↓                                                   │
│     Tải providers, modules, khởi tạo backend            │
│                                                         │
│  ② terraform validate / fmt (tùy chọn nhưng nên dùng)  │
│     ↓                                                   │
│     Kiểm tra cú pháp và format code                    │
│                                                         │
│  ③ terraform plan                                       │
│     ↓                                                   │
│     So sánh desired state vs current state              │
│     Tạo execution plan — không thay đổi gì              │
│                                                         │
│  ④ [Review plan output carefully!]                      │
│     ↓                                                   │
│     Con người đọc và xác nhận plan                      │
│                                                         │
│  ⑤ terraform apply                                      │
│     ↓                                                   │
│     Thực thi plan, gọi cloud APIs, cập nhật state       │
│                                                         │
│  ⑥ terraform destroy (khi cần dọn dẹp)                 │
│     ↓                                                   │
│     Xóa tất cả resources được quản lý                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. terraform init — Khởi Tạo

### Mục Đích

`terraform init` chuẩn bị working directory — thư mục làm việc — để dùng Terraform. Phải chạy trước tiên khi:
- Clone repo — Sao chép kho mã — mới
- Thêm/thay đổi provider
- Thêm/thay đổi module
- Thay đổi backend configuration

### Điều Gì Xảy Ra Khi Chạy init?

```bash
$ terraform init

Initializing the backend...
# 1. Khởi tạo backend (S3, GCS, local...) để lưu state

Initializing provider plugins...
# 2. Tải provider plugins từ Terraform Registry (hoặc private registry)
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Initializing modules...
# 3. Tải module sources (git, registry, local path)
- Downloading terraform-aws-modules/vpc/aws 5.4.0...

Terraform has been successfully initialized!
```

**Sau khi init, thư mục `.terraform/` được tạo:**

```
.terraform/
├── providers/
│   └── registry.terraform.io/
│       └── hashicorp/
│           └── aws/
│               └── 5.31.0/
│                   └── linux_amd64/
│                       └── terraform-provider-aws_v5.31.0  # Binary plugin
└── modules/
    └── vpc/            # Module đã tải
        └── ...

.terraform.lock.hcl     # Lock file — ghi lại chính xác version đang dùng
```

### Tùy Chọn Quan Trọng

```bash
# Nâng cấp provider/module lên version mới nhất (trong constraints)
terraform init -upgrade

# Không kết nối internet (dùng providers đã cache)
terraform init -plugin-dir=/path/to/plugins

# Migrate backend — Di chuyển sang backend mới
terraform init -migrate-state

# Reconfigure backend không hỏi lại
terraform init -reconfigure

# Chỉ tải modules (không tải providers)
terraform get
```

### Lock File — File Khóa Phiên Bản

```hcl
# .terraform.lock.hcl — TẠO TỰ ĐỘNG, KHÔNG SỬA TAY
# PHẢI commit file này vào git!

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",  # Hash của binary để verify integrity
    "zh:def456...",
  ]
}
```

**Tại sao commit lock file?**
- Đảm bảo tất cả team members và CI/CD dùng cùng provider version
- Tránh "works on my machine" problems
- Audit trail — Dấu vết kiểm tra — khi nâng cấp provider

---

## 3. terraform plan — Lập Kế Hoạch

### Mục Đích

`terraform plan` tạo execution plan — kế hoạch thực thi — cho thấy Terraform **sẽ làm gì** mà **không thực sự làm gì**. Đây là bước review — xem xét — quan trọng nhất.

### Điều Gì Xảy Ra Khi Chạy plan?

```
terraform plan

  Bước 1: Đọc current state
     ↓ Đọc terraform.tfstate (local hoặc remote backend)
     ↓ Gọi provider APIs để verify resource còn tồn tại
       (Refresh — Làm mới state)

  Bước 2: Load desired state
     ↓ Đọc tất cả .tf files
     ↓ Evaluate expressions, resolve references
     ↓ Build resource graph — Đồ thị tài nguyên

  Bước 3: Diff — So sánh
     ↓ Current state vs Desired state
     ↓ Quyết định: create / update / replace / delete / no-op

  Bước 4: Output plan
     ↓ Hiển thị chi tiết những gì sẽ thay đổi
```

### Đọc Plan Output — Đầu Ra Plan

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create        # Tạo mới
  ~ update        # Cập nhật (in-place)
  - destroy       # Xóa
  -/+ destroy and then create (replace)   # Xóa rồi tạo lại
  +/- create and then destroy (replace)   # Tạo rồi xóa (với create_before_destroy)
  <= read         # Chỉ đọc (data source)

# Ví dụ output:
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + arn                          = (known after apply)     # Chỉ biết sau khi tạo
      + id                           = (known after apply)
      + instance_type                = "t3.micro"
      + public_ip                    = (known after apply)
      + tags                         = {
          + "Environment" = "dev"
          + "Name"        = "myapp-dev-web"
        }
    }

  # aws_db_instance.main will be updated in-place
  ~ resource "aws_db_instance" "main" {
        id                    = "myapp-dev-db"
      ~ backup_retention_period = 7 -> 14   # Giá trị thay đổi
        # (các attributes khác không thay đổi)
    }

  # aws_security_group.old will be destroyed
  - resource "aws_security_group" "old" {
      - id   = "sg-0abc123"
      - name = "old-sg"
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

### Ký Hiệu Quan Trọng Trong Plan

| Ký Hiệu | Ý Nghĩa | Hành Động |
|---------|---------|----------|
| `+` | create | Tạo resource mới |
| `~` | update | Cập nhật resource (in-place — tại chỗ) |
| `-` | destroy | Xóa resource |
| `-/+` | replace | Xóa resource cũ → Tạo resource mới (gây downtime!) |
| `+/-` | replace | Tạo resource mới → Xóa resource cũ (với `create_before_destroy`) |
| `<=` | read | Đọc data source |

**Cảnh báo đỏ:** `-/+` (replace) có nghĩa là resource sẽ bị xóa và tạo lại. Với database → mất data. Với EC2 → downtime. Phải review cẩn thận!

### Tùy Chọn Plan Quan Trọng

```bash
# Lưu plan vào file để apply sau (không hỏi lại)
terraform plan -out=tfplan

# Apply plan đã lưu (không cần confirm)
terraform apply tfplan

# Chỉ plan cho một resource cụ thể
terraform plan -target=aws_instance.web

# Plan với variable cụ thể
terraform plan -var-file=prod.tfvars

# Plan với refresh bị tắt (nhanh hơn nhưng không refresh state)
terraform plan -refresh=false

# Destroy plan — Xem trước khi destroy
terraform plan -destroy
```

### Tại Sao Phải Đọc Plan Trước Khi Apply?

```bash
# Scenario — Kịch bản nguy hiểm:
# Bạn đổi tên S3 bucket trong code
# Old: bucket = "myapp-prod-data"
# New: bucket = "myapp-prod-storage"

# Plan output:
  - aws_s3_bucket.data will be destroyed    # ← XÓA bucket cũ với TẤT CẢ DATA!
  + aws_s3_bucket.data will be created      # ← Tạo bucket mới (rỗng)

# Nếu không đọc plan và cứ apply → MẤT DATA PRODUCTION!
# Giải pháp đúng: Dùng "moved" block để rename mà không recreate
```

---

## 4. terraform apply — Áp Dụng

### Mục Đích

`terraform apply` thực thi execution plan, gọi cloud APIs để tạo/cập nhật/xóa resources, và cập nhật state file.

### Quy Trình Apply

```
terraform apply

  1. Chạy lại plan (để đảm bảo up-to-date)
  2. Hiển thị plan
  3. Hỏi xác nhận: "Do you want to perform these actions? [yes/no]"
  4. Sau khi gõ "yes":
     - Gọi cloud APIs theo dependency order
     - Cập nhật state file sau mỗi resource thành công
     - Hiển thị progress — Tiến trình
  5. Hiển thị outputs nếu có
```

### Tùy Chọn Apply Quan Trọng

```bash
# Apply plan đã lưu (recommended cho production!)
terraform apply tfplan

# Auto-approve — Không hỏi xác nhận (dùng trong CI/CD)
terraform apply -auto-approve

# Chỉ apply cho resource cụ thể (nguy hiểm, dùng cẩn thận)
terraform apply -target=aws_instance.web

# Apply với parallelism — Số resources xử lý song song (default: 10)
terraform apply -parallelism=5

# Apply với biến cụ thể
terraform apply -var-file=prod.tfvars -auto-approve
```

### Apply Với Plan File — Cách Tốt Nhất Cho Production

```bash
# Step 1: Tạo và review plan
terraform plan -out=tfplan.binary

# Step 2: Xem plan (human-readable)
terraform show tfplan.binary

# Step 3: Apply plan đã review
terraform apply tfplan.binary
# Lần này không hỏi xác nhận vì đã review plan rồi
# Đảm bảo apply đúng những gì đã review
```

**Tại sao quan trọng?** Nếu chạy `terraform apply` không có plan file, Terraform sẽ tạo plan mới tại thời điểm apply. Nếu state thay đổi giữa lúc plan và apply (do người khác apply), bạn có thể apply nhầm.

### Hiểu Output Của Apply

```bash
aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Still creating... [20s elapsed]
aws_instance.web: Creation complete after 23s [id=i-0abc123def456]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_public_ip = "52.77.123.45"
instance_id = "i-0abc123def456"
```

### Partial Apply — Apply Một Phần

```bash
# Nếu apply fail giữa chừng:
# - Resources đã tạo xong → ghi vào state, giữ nguyên
# - Resources chưa tạo → không có trong state
# → Chạy lại apply để tiếp tục từ chỗ bỏ dở

terraform apply  # Chạy lại an toàn — idempotent
```

---

## 5. terraform destroy — Hủy

### Mục Đích

`terraform destroy` xóa **tất cả** resources được quản lý bởi Terraform configuration hiện tại. Thường dùng cho:
- Dọn dẹp môi trường dev/test
- Xóa môi trường không còn cần thiết
- Chi phí: teardown sau giờ làm việc

### Cách Dùng

```bash
# Xem trước sẽ xóa gì
terraform plan -destroy

# Destroy với xác nhận
terraform destroy

# Destroy không hỏi (dùng trong CI/CD cho dev env)
terraform destroy -auto-approve

# Chỉ destroy resource cụ thể (NGUY HIỂM - dễ ảnh hưởng dependencies)
terraform destroy -target=aws_instance.web
```

### Thứ Tự Xóa

Terraform xóa resources theo thứ tự **ngược với khi tạo** — reverse dependency order:

```
Khi tạo:
  aws_vpc → aws_subnet → aws_instance → aws_eip

Khi xóa:
  aws_eip → aws_instance → aws_subnet → aws_vpc
  (phải xóa các resources phụ thuộc trước)
```

### Bảo Vệ Resources Quan Trọng

```hcl
# Ngăn terraform destroy xóa resource này
resource "aws_db_instance" "production" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}

# Kết quả khi cố destroy:
# Error: Instance cannot be destroyed
# Resource aws_db_instance.production has lifecycle.prevent_destroy set to true.
```

### Destroy Có Chọn Lựa (Không Dùng -target)

```bash
# Cách an toàn hơn -target:
# Xóa resource khỏi state mà không xóa trên cloud
terraform state rm aws_instance.web

# Sau đó muốn Terraform quản lý lại:
terraform import aws_instance.web i-0abc123def456
```

---

## 6. Các Lệnh Hỗ Trợ Quan Trọng

### terraform fmt — Format Code

```bash
# Format tất cả .tf files trong thư mục hiện tại
terraform fmt

# Format đệ quy — recursive — cả subfolders
terraform fmt -recursive

# Chỉ kiểm tra (không sửa), exit code 1 nếu cần format
terraform fmt -check  # Dùng trong CI/CD

# Xem diff — Sự khác biệt — trước khi sửa
terraform fmt -diff
```

### terraform validate — Kiểm Tra Cú Pháp

```bash
# Kiểm tra syntax và consistency (không cần credentials)
terraform validate

# Output:
# Success! The configuration is valid.
# hoặc
# Error: Reference to undeclared resource
```

**Lưu ý:** `validate` chỉ kiểm tra syntax — cú pháp — không kiểm tra logic hay resource values thực tế.

### terraform show — Xem State/Plan

```bash
# Xem toàn bộ current state (human-readable)
terraform show

# Xem state dưới dạng JSON
terraform show -json

# Xem plan file đã lưu
terraform show tfplan.binary
```

### terraform output — Xem Outputs

```bash
terraform output                    # Tất cả outputs
terraform output vpc_id             # Output cụ thể
terraform output -raw vpc_id        # Không có dấu nháy
terraform output -json              # Dạng JSON
```

### terraform refresh — Làm Mới State

```bash
# Cập nhật state để phản ánh actual cloud state
# (Đã được tích hợp vào plan bởi default)
terraform refresh

# Lưu ý: Trong Terraform >= 0.15, dùng:
terraform apply -refresh-only
# Chỉ cập nhật state, không apply changes
```

### terraform state — Quản Lý State Trực Tiếp

```bash
# Liệt kê tất cả resources trong state
terraform state list

# Xem chi tiết một resource trong state
terraform state show aws_instance.web

# Xóa resource khỏi state (không xóa trên cloud)
terraform state rm aws_instance.web

# Di chuyển resource trong state (đổi tên không tạo lại)
terraform state mv aws_instance.web aws_instance.web_server

# Xem state file raw
terraform state pull

# Ghi đè state file (nguy hiểm!)
terraform state push
```

### terraform import — Import Resource Hiện Có

```bash
# Import EC2 instance đã tạo tay vào Terraform management
terraform import aws_instance.web i-0abc123def456

# Import với config file (Terraform >= 1.5)
# import block trong .tf file:
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}

# Sau đó chạy:
terraform plan  # Terraform generate config suggestion
```

### terraform workspace — Quản Lý Workspace

```bash
terraform workspace list            # Liệt kê workspaces
terraform workspace new dev         # Tạo workspace mới
terraform workspace select prod     # Chuyển sang workspace
terraform workspace show            # Workspace hiện tại
terraform workspace delete dev      # Xóa workspace
```

---

## 7. Terraform Workflow Trong Team

### Workflow Đơn Giản (Team Nhỏ)

```bash
# Developer A
git checkout -b feature/add-rds
# ... sửa Terraform code ...
git push origin feature/add-rds

# CI/CD Pipeline tự động chạy:
terraform fmt -check     # Kiểm tra format
terraform validate       # Kiểm tra syntax
terraform plan           # Post plan output lên PR comment

# Developer B review PR + plan output
# Sau khi approve → merge

# CI/CD Pipeline chạy trên main:
terraform apply -auto-approve
```

### Workflow Chuẩn (Team Lớn / Production)

```
┌─────────────────────────────────────────────────────────┐
│                   GitOps Terraform Workflow              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Dev tạo feature branch                             │
│     git checkout -b feature/add-redis                  │
│                                                         │
│  2. Dev viết Terraform code                            │
│                                                         │
│  3. Dev tạo PR — Pull Request                          │
│     CI tự động chạy:                                   │
│     - terraform fmt -check                             │
│     - terraform validate                               │
│     - tflint (linting)                                 │
│     - tfsec / checkov (security scanning)             │
│     - terraform plan → Post output vào PR comment      │
│     - infracost (cost estimate — ước tính chi phí)     │
│                                                         │
│  4. Team review PR + plan output                       │
│     Chú ý: destroy actions, replace actions            │
│                                                         │
│  5. Sau khi approve và merge vào main                  │
│     CI tự động chạy:                                   │
│     - terraform plan -out=tfplan                       │
│     - [Optional: Manual approval gate]                 │
│     - terraform apply tfplan                           │
│                                                         │
│  6. Monitor kết quả                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Locking Trong Team

```hcl
# Khi dùng remote backend với locking, chỉ một người apply cùng lúc
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-state-lock"  # DynamoDB cho locking
    encrypt        = true
  }
}

# Nếu người khác đang apply:
# Error: Error acquiring the state lock
# Lock Info:
#   ID: 12345678-1234-1234-1234-123456789012
#   Path: s3://myapp-terraform-state/prod/terraform.tfstate
#   Operation: OperationTypeApply
#   Who: user@machine
#   Created: 2026-05-12 10:30:00 +0000 UTC
```

---

## 8. Xử Lý Lỗi Thường Gặp

### Lỗi "Error acquiring the state lock"

```bash
# Khi apply bị interrupt — ngắt ngang — lock không được release
# Xem thông tin lock
terraform force-unlock <LOCK_ID>

# Cẩn thận: Chỉ dùng khi chắc chắn không ai đang apply!
```

### Lỗi Provider Authentication

```bash
# Lỗi: No valid credential sources found
# Giải pháp:
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="ap-southeast-1"

# Hoặc đảm bảo AWS profile được cấu hình:
aws configure list
```

### Lỗi "Cycle detected in dependency graph"

```
Error: Cycle: aws_security_group.a → aws_security_group.b → aws_security_group.a
```

```hcl
# Giải pháp: Tách rules ra khỏi security group
resource "aws_security_group" "a" {
  name = "sg-a"
  # Không đặt ingress/egress inline
}

resource "aws_security_group" "b" {
  name = "sg-b"
}

resource "aws_security_group_rule" "a_to_b" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.a.id
  security_group_id        = aws_security_group.b.id
}
```

### Lỗi "Provider produced inconsistent result after apply"

```
Error: Provider produced inconsistent result after apply
```

Thường do provider bug hoặc API eventually consistent — nhất quán cuối cùng. Giải pháp:
1. Nâng cấp provider version
2. Thêm `time_sleep` resource nếu API cần thời gian
3. Report bug lên provider GitHub

### Debug Mode — Chế Độ Gỡ Lỗi

```bash
# Bật debug logging — Ghi nhật ký gỡ lỗi
export TF_LOG=DEBUG
terraform apply 2> terraform-debug.log

# Các mức độ log:
# TRACE  — Chi tiết nhất
# DEBUG  — Debug thông thường
# INFO   — Thông tin chung
# WARN   — Cảnh báo
# ERROR  — Chỉ lỗi

# Log ra file cụ thể
export TF_LOG_PATH="./terraform.log"
export TF_LOG=DEBUG
terraform plan
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Giải thích toàn bộ vòng đời Terraform từ `init` đến `apply`.**

> `terraform init` chuẩn bị working directory: tải provider plugins từ Terraform Registry, tải modules, và khởi tạo backend để lưu state. `terraform plan` so sánh desired state — trạng thái mong muốn — từ .tf files với current state từ state file và actual cloud state, tạo execution plan — kế hoạch thực thi — cho thấy sẽ create/update/delete gì mà không làm gì thực. `terraform apply` thực thi plan, gọi cloud APIs để tạo/cập nhật/xóa resources theo dependency order — thứ tự phụ thuộc, cập nhật state file sau mỗi bước.

**Q: Tại sao nên dùng `terraform plan -out=tfplan` thay vì chỉ chạy `terraform apply`?**

> Khi lưu plan vào file và apply file đó, Terraform đảm bảo apply đúng những gì đã review — xem xét. Nếu chạy `terraform apply` không có plan file, Terraform tạo plan mới lúc apply, có thể khác với plan đã review nếu ai đó thay đổi state hoặc cloud infrastructure trong khoảng thời gian giữa plan và apply. Trong CI/CD cho production, đây là best practice — thực hành tốt nhất — quan trọng.

**Q: `terraform refresh` và `terraform apply -refresh-only` khác nhau thế nào?**

> Cả hai đều cập nhật state file để phản ánh actual cloud state mà không apply configuration changes. Tuy nhiên, `terraform refresh` là deprecated — không còn được khuyến dùng — từ Terraform 0.15 vì nó cập nhật state mà không có approval step — bước xác nhận. `terraform apply -refresh-only` tạo plan hiển thị những gì sẽ thay đổi trong state, cho phép review trước khi confirm. An toàn hơn vì vẫn có approval step.

**Q: Khi apply bị interrupt — ngắt ngang — giữa chừng, Terraform xử lý thế nào?**

> Terraform tính toán resources theo thứ tự và cập nhật state file ngay sau khi mỗi resource hoàn thành. Nếu bị interrupt, state file phản ánh đúng những gì đã apply thành công. Chạy lại `terraform apply` là an toàn — idempotent — sẽ chỉ tạo/cập nhật những resources chưa hoàn thành. Tuy nhiên, nếu bị interrupt trong khi đang tạo một resource, resource đó có thể tồn tại trên cloud nhưng chưa vào state → orphaned resource — tài nguyên mồ côi. Terraform sẽ cố gắng tạo lại và gặp conflict. Cần dùng `terraform import` để đưa resource vào state.

**Q: Sự khác biệt giữa `terraform destroy -target` và `terraform state rm`?**

> `terraform destroy -target=<resource>` xóa resource trên cloud AND xóa khỏi state file. `terraform state rm <resource>` chỉ xóa resource khỏi state file, resource vẫn còn tồn tại trên cloud. Dùng `state rm` khi muốn Terraform "quên" resource (ví dụ: để import lại sau). Dùng `destroy -target` khi thực sự muốn xóa resource, nhưng cẩn thận với dependencies.

---

## 📋 Checklist

- [ ] Chạy được `terraform init`, `plan`, `apply`, `destroy` thành công
- [ ] Đọc và hiểu plan output trước khi confirm apply
- [ ] Biết ký hiệu `+`, `~`, `-`, `-/+` có nghĩa gì
- [ ] Dùng `terraform plan -out=tfplan` và `terraform apply tfplan`
- [ ] Cấu hình TF_LOG để debug khi cần
- [ ] Biết các lệnh state management cơ bản

---

## 🔗 Liên Kết

- ← [4. Variables & Outputs](./4-variables-outputs.md)
- → [02 State Management](../02-state-management/README.md) — Phần tiếp theo
- [README phần Fundamentals](./README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
