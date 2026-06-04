# terraform import & moved block — Quản Lý Tài Nguyên Hiện Có

> Hai kỹ thuật thiết yếu khi làm việc với hạ tầng thực tế: `terraform import` để đưa resource bên ngoài vào quản lý, và `moved` block để refactor code mà không rebuild infrastructure.

---

## Vấn Đề Cần Giải Quyết

### Tình Huống 1: Resource Đã Tồn Tại Bên Ngoài Terraform

```
Thực tế:         S3 bucket "prod-data-2019" đang chạy
Terraform state: Không biết bucket này tồn tại

→ Cần: Đưa bucket vào quản lý Terraform mà không xóa và tạo lại
→ Giải pháp: terraform import
```

### Tình Huống 2: Refactor Code Mà Không Rebuild

```
Trước:  resource "aws_s3_bucket" "data" { ... }
Sau:    resource "aws_s3_bucket" "prod_data" { ... }  ← Đổi tên

Vấn đề: Terraform hiểu là xóa "data" và tạo "prod_data" mới
Thực tế mong muốn: Chỉ đổi tên trong code, bucket không bị xóa

→ Giải pháp: moved block
```

---

## terraform import — Đưa Resource Vào Quản Lý

### Cú Pháp Cơ Bản

```bash
# Cú pháp
terraform import <RESOURCE_TYPE>.<RESOURCE_NAME> <RESOURCE_ID>

# Ví dụ thực tế
terraform import aws_s3_bucket.data prod-data-2019
terraform import aws_instance.web i-0abc123def456789a
terraform import aws_db_instance.postgres my-production-db
terraform import aws_iam_role.lambda arn:aws:iam::123456789012:role/LambdaRole
terraform import aws_vpc.main vpc-0123456789abcdef0
```

### Quy Trình Import Đúng

```bash
# Bước 1: Viết resource block trong code (cần thiết trước khi import)
# main.tf
resource "aws_s3_bucket" "data" {
  bucket = "prod-data-2019"
  # Không cần điền đầy đủ — Terraform sẽ populate sau khi import
}

# Bước 2: Chạy import
terraform import aws_s3_bucket.data prod-data-2019

# Bước 3: Xem state đã được populate
terraform state show aws_s3_bucket.data

# Bước 4: Chạy plan để xem sự khác biệt
terraform plan
# Plan sẽ hiển thị những attribute code chưa khai báo

# Bước 5: Cập nhật code để match với state
# Thêm vào main.tf những attribute plan báo sẽ thay đổi

# Bước 6: Verify plan không còn thay đổi
terraform plan
# Phải hiển thị: "No changes. Infrastructure is up-to-date."
```

### Import IDs Của Các Resource Phổ Biến

| Resource                              | Import ID Format                                           |
| ------------------------------------- | ---------------------------------------------------------- |
| `aws_instance`                        | Instance ID: `i-0abc123def456789a`                        |
| `aws_s3_bucket`                       | Bucket name: `my-bucket-name`                             |
| `aws_db_instance`                     | DB identifier: `my-production-db`                         |
| `aws_vpc`                             | VPC ID: `vpc-0123456789abcdef0`                           |
| `aws_security_group`                  | SG ID: `sg-0abc123def456789a`                             |
| `aws_iam_role`                        | Role ARN hoặc name: `my-role-name`                        |
| `aws_iam_policy`                      | Policy ARN: `arn:aws:iam::123:policy/MyPolicy`            |
| `aws_route53_record`                  | `ZONE_ID_RECORD_NAME_TYPE`: `Z123_example.com_A`          |
| `google_compute_instance`             | `projects/PROJECT/zones/ZONE/instances/NAME`              |
| `azurerm_resource_group`              | `/subscriptions/SUB_ID/resourceGroups/GROUP_NAME`         |

### Import Nhiều Resource Cùng Lúc (Terraform 1.5+)

Từ Terraform 1.5, có thể dùng `import` block trong code:

```hcl
# import.tf — Khai báo import như code
import {
  to = aws_s3_bucket.data
  id = "prod-data-2019"
}

import {
  to = aws_instance.web
  id = "i-0abc123def456789a"
}

import {
  to = aws_db_instance.postgres
  id = "my-production-db"
}
```

```bash
# Chạy plan với import blocks
terraform plan
# Terraform sẽ import và hiển thị kết quả

# Sau khi apply xong, xóa import blocks khỏi code
# (Chúng chỉ cần chạy một lần)
```

### Generate Config — Tự Động Tạo Code (Terraform 1.5+)

```bash
# Tự động generate HCL code từ resource đang import
terraform plan -generate-config-out=generated.tf

# generated.tf sẽ chứa đầy đủ attribute của resource
# Review và cleanup file này trước khi dùng
```

---

## moved Block — Di Chuyển Resource Mà Không Rebuild

### Mục Đích Của moved Block

`moved` block cho Terraform biết: "Resource này đã di chuyển địa chỉ trong state, nhưng tài nguyên thực tế KHÔNG thay đổi."

```hcl
# Cú pháp
moved {
  from = <địa_chỉ_cũ>
  to   = <địa_chỉ_mới>
}
```

### Ví Dụ 1: Đổi Tên Resource

```hcl
# Trước (code cũ):
resource "aws_s3_bucket" "data" {
  bucket = "prod-data-2019"
}

# Sau khi refactor (code mới):
resource "aws_s3_bucket" "prod_data" {    # ← Đổi tên
  bucket = "prod-data-2019"
}

# moved block để Terraform KHÔNG xóa và tạo lại
moved {
  from = aws_s3_bucket.data
  to   = aws_s3_bucket.prod_data
}
```

### Ví Dụ 2: Di Chuyển Vào Module

```hcl
# Trước: Resource ở root module
resource "aws_instance" "web" { ... }

# Sau: Di chuyển vào module
module "web_server" {
  source = "./modules/web-server"
  # ...
}
# Trong module, resource tên là aws_instance.main

# moved block
moved {
  from = aws_instance.web
  to   = module.web_server.aws_instance.main
}
```

### Ví Dụ 3: Chuyển Từ count Sang for\_each

```hcl
# Trước: Dùng count — tạo resource theo số lượng
resource "aws_instance" "server" {
  count = 3
  # ...
}
# State address: aws_instance.server[0], [1], [2]

# Sau: Chuyển sang for_each — tạo resource theo map
resource "aws_instance" "server" {
  for_each = toset(["web", "api", "worker"])
  # ...
}
# State address: aws_instance.server["web"], ["api"], ["worker"]

# moved blocks để map từng instance
moved {
  from = aws_instance.server[0]
  to   = aws_instance.server["web"]
}

moved {
  from = aws_instance.server[1]
  to   = aws_instance.server["api"]
}

moved {
  from = aws_instance.server[2]
  to   = aws_instance.server["worker"]
}
```

### Ví Dụ 4: Chuyển Resource Vào Module Khác

```hcl
# Di chuyển giữa modules
moved {
  from = module.old_module.aws_security_group.web
  to   = module.new_module.aws_security_group.web
}
```

---

## So Sánh import vs moved vs terraform state mv

| Tình Huống                                          | Giải Pháp                     |
| --------------------------------------------------- | ----------------------------- |
| Resource tồn tại bên ngoài Terraform                | `terraform import`            |
| Đổi tên/di chuyển resource trong code               | `moved` block                 |
| Di chuyển resource trong state (không qua code)     | `terraform state mv`          |
| Tách state sang workspace khác                      | `terraform state mv` + `push` |

### Khi Nào Dùng `terraform state mv`?

```bash
# terraform state mv: Thao tác trực tiếp trên state
# Dùng khi không muốn hoặc không thể dùng moved block trong code

# Ví dụ: Di chuyển resource giữa modules
terraform state mv \
  'module.old.aws_instance.web' \
  'module.new.aws_instance.web'

# Ví dụ: Đổi tên resource
terraform state mv \
  'aws_s3_bucket.old_name' \
  'aws_s3_bucket.new_name'
```

**Khác biệt quan trọng:**
- `moved` block: Declarative — Khai báo — có trong version control, được review qua PR
- `terraform state mv`: Imperative — Mệnh lệnh — chạy một lần, không track trong code

**Ưu tiên dùng `moved` block** vì nó an toàn hơn, có thể review, và self-documenting.

---

## Best Practices — Thực Hành Tốt Nhất

### Với terraform import

```bash
# 1. Luôn backup state trước khi import
terraform state pull > backup-$(date +%Y%m%d).json

# 2. Import trong môi trường dev/staging trước
# Sau đó sao chép quy trình sang prod

# 3. Verify plan = "No changes" sau khi import
# Nếu plan có changes: update code để match với thực tế

# 4. Không import nhiều resource cùng lúc nếu không chắc
# Import từng cái và verify từng cái
```

### Với moved Block

```hcl
# 1. Giữ moved blocks tạm thời — xóa sau 1-2 sprint
# Sau khi team đã apply ở tất cả environment

# 2. Comment khi nào moved block được tạo
moved {
  from = aws_s3_bucket.data      # Đổi tên: 2026-05-12
  to   = aws_s3_bucket.prod_data
}

# 3. Không xóa moved block trước khi apply ở TẤT CẢ environment
# Nếu xóa sớm: người apply sau sẽ bị Terraform xóa và tạo lại resource
```

---

## Câu Hỏi Phỏng Vấn

**Q: Làm sao đưa tài nguyên đang chạy vào quản lý bởi Terraform?**

A: Dùng `terraform import`. Quy trình:
1. Viết resource block trong code
2. Chạy `terraform import <resource_address> <resource_id>`
3. Chạy `terraform state show` để xem state đã populate
4. Cập nhật code để match với state
5. Verify `terraform plan` không hiển thị unexpected changes

Từ Terraform 1.5 có thể dùng `import` block trong code và `terraform plan -generate-config-out` để tự động tạo code.

**Q: `moved` block dùng khi nào?**

A: Khi refactor code Terraform mà không muốn Terraform xóa và tạo lại tài nguyên. Ví dụ: đổi tên resource, di chuyển resource vào module, chuyển từ `count` sang `for_each`. `moved` block tốt hơn `terraform state mv` vì nó được track trong version control và có thể review qua PR.

---

## Tóm Tắt

```
terraform import:
├── Khi resource tồn tại bên ngoài Terraform
├── Quy trình: Write code → Import → Verify plan = No changes
└── Terraform 1.5+: Dùng import block + -generate-config-out

moved block:
├── Khi refactor code mà không muốn rebuild resource
├── Ưu tiên hơn terraform state mv vì declarative và reviewable
└── Xóa moved block sau khi apply ở tất cả environment

terraform state mv:
└── Thao tác state trực tiếp khi không muốn dùng moved block trong code
```

---

**Xem Thêm:**
- [`1-state-corruption.md`](./1-state-corruption.md) — Phục hồi state khi có vấn đề
- [`02-state-management/4-state-commands.md`](../02-state-management/4-state-commands.md) — Tất cả state commands
