# 4 — terraform state Commands — Các Lệnh Thao Tác Trạng Thái

> Các lệnh `terraform state` là bộ công cụ phẫu thuật hạ tầng — dùng để quản trị, di chuyển, và sửa chữa state mà không cần destroy và tạo lại tài nguyên.

---

## 📚 Mục Lục

1. [Tổng Quan Các Lệnh](#1-tổng-quan-các-lệnh)
2. [terraform state list — Liệt Kê Tài Nguyên](#2-terraform-state-list--liệt-kê-tài-nguyên)
3. [terraform state show — Xem Chi Tiết](#3-terraform-state-show--xem-chi-tiết)
4. [terraform state mv — Di Chuyển Tài Nguyên](#4-terraform-state-mv--di-chuyển-tài-nguyên)
5. [terraform state rm — Xóa Khỏi State](#5-terraform-state-rm--xóa-khỏi-state)
6. [terraform state pull — Tải State Xuống](#6-terraform-state-pull--tải-state-xuống)
7. [terraform state push — Đẩy State Lên](#7-terraform-state-push--đẩy-state-lên)
8. [terraform import — Import Tài Nguyên Hiện Có](#8-terraform-import--import-tài-nguyên-hiện-có)
9. [moved Block — Di Chuyển Trong Code](#9-moved-block--di-chuyển-trong-code)
10. [Workflow Thực Tế — Các Kịch Bản Hay Gặp](#10-workflow-thực-tế--các-kịch-bản-hay-gặp)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Các Lệnh

| Lệnh | Mục Đích | Thay Đổi Cloud? | Nguy Hiểm? |
|------|---------|-----------------|------------|
| `state list` | Liệt kê tất cả resources trong state | Không | Không |
| `state show` | Xem thuộc tính chi tiết của một resource | Không | Không |
| `state mv` | Di chuyển/đổi tên resource trong state | Không | Trung bình |
| `state rm` | Xóa resource khỏi state (giữ nguyên trên cloud) | Không | Cao |
| `state pull` | Tải state file xuống stdout | Không | Không |
| `state push` | Đẩy state file lên backend | Không | Rất cao |
| `import` | Đưa resource hiện có vào Terraform management | Không | Trung bình |

**Lưu ý quan trọng:** Tất cả các lệnh trên thay đổi state file nhưng **không thay đổi hạ tầng thực tế** (trừ `import` trong một số trường hợp đặc biệt). Luôn backup state trước khi thực hiện.

---

## 2. terraform state list — Liệt Kê Tài Nguyên

```bash
# Liệt kê tất cả resources
terraform state list

# Output ví dụ:
aws_instance.web
aws_instance.worker[0]
aws_instance.worker[1]
aws_security_group.web_sg
aws_lb.main
module.vpc.aws_vpc.main
module.vpc.aws_subnet.private[0]
module.vpc.aws_subnet.private[1]
module.vpc.aws_subnet.public[0]
```

**Lọc theo pattern:**

```bash
# Chỉ hiện resources trong module vpc
terraform state list module.vpc

# Chỉ hiện aws_instance resources
terraform state list 'aws_instance.*'

# Xem resource trong module lồng nhau — nested module
terraform state list module.eks.module.node_group
```

**Khi nào dùng:**
- Kiểm tra Terraform đang quản lý những gì
- Tìm tên chính xác của resource trước khi dùng `state show` hoặc `state mv`
- Debug khi plan không như mong đợi

---

## 3. terraform state show — Xem Chi Tiết

```bash
# Xem tất cả thuộc tính của một resource
terraform state show aws_instance.web

# Output ví dụ:
# aws_instance.web:
resource "aws_instance" "web" {
    ami                         = "ami-0c55b159cbfafe1f0"
    arn                         = "arn:aws:ec2:us-east-1:123456789:instance/i-0abc123def"
    id                          = "i-0abc123def456789"
    instance_type               = "t3.micro"
    private_ip                  = "10.0.1.15"
    public_ip                   = "54.12.34.56"
    tags                        = {
        "Environment" = "production"
        "Name"        = "web-server"
    }
    # ... nhiều thuộc tính khác
}
```

```bash
# Xem resource trong module
terraform state show 'module.vpc.aws_vpc.main'

# Xem resource được tạo bằng count
terraform state show 'aws_instance.worker[0]'

# Xem resource được tạo bằng for_each
terraform state show 'aws_instance.worker["us-east-1a"]'
```

**Khi nào dùng:**
- Tìm ID thực tế của resource để dùng trong script
- Debug khi Terraform plan muốn thay đổi giá trị không mong muốn
- Xác minh thuộc tính sau khi import

---

## 4. terraform state mv — Di Chuyển Tài Nguyên

Đổi tên hoặc di chuyển resource trong state mà không destroy và recreate — tái tạo — tài nguyên.

### Đổi tên resource

```bash
# Đổi tên từ "web" thành "web_server"
terraform state mv aws_instance.web aws_instance.web_server
```

```hcl
# Phải cập nhật code tương ứng:
# Trước:
resource "aws_instance" "web" { ... }

# Sau:
resource "aws_instance" "web_server" { ... }
```

### Di chuyển vào module

```bash
# Di chuyển resource vào module
terraform state mv aws_instance.web module.web_tier.aws_instance.main
```

### Di chuyển từ count sang for_each — Trường hợp phổ biến

```bash
# Trước: aws_instance.worker[0], aws_instance.worker[1]
# Sau refactor sang for_each với keys "primary", "secondary"

terraform state mv 'aws_instance.worker[0]' 'aws_instance.worker["primary"]'
terraform state mv 'aws_instance.worker[1]' 'aws_instance.worker["secondary"]'
```

### Di chuyển giữa các state file (cross-state)

```bash
# Di chuyển resource từ state của module A sang module B
terraform state mv \
  -state=./module-a/terraform.tfstate \
  -state-out=./module-b/terraform.tfstate \
  aws_instance.shared \
  aws_instance.shared
```

**Quy trình an toàn khi dùng `state mv`:**

```bash
# 1. Backup state trước
terraform state pull > backup_$(date +%Y%m%d_%H%M%S).tfstate

# 2. Thực hiện mv
terraform state mv aws_instance.web aws_instance.web_server

# 3. Cập nhật code HCL tương ứng

# 4. Verify bằng plan — phải hiện "No changes"
terraform plan
```

---

## 5. terraform state rm — Xóa Khỏi State

Xóa resource khỏi state file nhưng **giữ nguyên tài nguyên thực tế trên cloud**. Terraform sẽ không quản lý tài nguyên đó nữa.

```bash
# Xóa một resource khỏi state
terraform state rm aws_instance.legacy

# Xóa toàn bộ module khỏi state
terraform state rm module.old_vpc
```

**Khi nào dùng `state rm`:**

```
1. Tài nguyên cần được quản lý bởi team/project khác
   → Xóa khỏi state hiện tại, import vào project mới

2. Tài nguyên cần tồn tại lâu dài nhưng không muốn destroy theo code
   → Ví dụ: S3 bucket lưu data lâu dài

3. Khắc phục khi state bị lỗi và cần clean up
   → Xóa resource lỗi, sau đó import lại

4. Unmanage — Không quản lý nữa — nhưng không xóa
```

**Ví dụ thực tế:**

```bash
# Kịch bản: Tách project lớn thành nhiều project nhỏ
# VPC sẽ được quản lý bởi project networking riêng

# 1. Xóa VPC resources khỏi project hiện tại
terraform state rm module.vpc.aws_vpc.main
terraform state rm module.vpc.aws_subnet.private

# 2. Trong project networking mới, import VPC
terraform import aws_vpc.main vpc-0abc123def
```

---

## 6. terraform state pull — Tải State Xuống

```bash
# Tải state hiện tại và in ra stdout
terraform state pull

# Lưu vào file để backup
terraform state pull > backup_$(date +%Y%m%d_%H%M%S).tfstate

# Xem state bằng jq — JSON query tool
terraform state pull | jq '.resources[].type' | sort | uniq -c

# Đếm số resources
terraform state pull | jq '.resources | length'

# Tìm resource theo type
terraform state pull | jq '.resources[] | select(.type == "aws_instance")'
```

**Khi nào dùng:**
- Backup state thủ công trước khi thực hiện thao tác nguy hiểm
- Inspect — Kiểm tra — state khi debugging
- Copy state từ backend này sang backend khác thủ công

---

## 7. terraform state push — Đẩy State Lên

```bash
# Push state file lên backend (thay thế state hiện tại!)
terraform state push terraform.tfstate.backup
```

**⚠️ CẢNH BÁO:** Đây là lệnh nguy hiểm nhất trong bộ state commands. Push state cũ lên sẽ làm mất mọi thay đổi sau thời điểm state đó được tạo.

**Chỉ dùng khi:**

```
1. Rollback — Quay lại — state sau khi apply bị lỗi
2. Phục hồi state từ backup sau khi corrupt
3. Di chuyển state giữa các backend thủ công (ưu tiên dùng terraform init -migrate-state)
```

**Quy trình an toàn:**

```bash
# 1. Luôn backup state hiện tại trước
terraform state pull > current_state_backup.tfstate

# 2. Verify nội dung file sẽ push
cat old_state.tfstate | jq '.serial'  # Serial phải cao hơn serial hiện tại

# 3. Push với xác nhận
terraform state push old_state.tfstate

# 4. Verify bằng plan ngay sau khi push
terraform plan
```

---

## 8. terraform import — Import Tài Nguyên Hiện Có

Đưa tài nguyên đã tồn tại trên cloud vào Terraform state để bắt đầu quản lý.

### Cú pháp cơ bản

```bash
# terraform import <resource_address> <resource_id>
terraform import aws_instance.web i-0abc123def456789
terraform import aws_s3_bucket.logs mycompany-logs-bucket
terraform import aws_route53_record.www Z1234567890ABC_example.com_A
```

### Import vào module

```bash
terraform import module.web_tier.aws_instance.main i-0abc123def456789
```

### Terraform v1.5+: Import Block — Khai Báo Import Trong Code

Terraform 1.5 giới thiệu `import` block — cú pháp khai báo trong HCL thay vì command line:

```hcl
# import.tf
import {
  to = aws_instance.web
  id = "i-0abc123def456789"
}

resource "aws_instance" "web" {
  # Terraform 1.5+ có thể tự generate config này:
  # terraform plan -generate-config-out=generated.tf
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

```bash
# Terraform tự generate code HCL cho resource
terraform plan -generate-config-out=generated_resources.tf

# Apply để thực hiện import
terraform apply
```

**Quy trình import thực tế:**

```bash
# 1. Viết resource block trong HCL (dù chưa đầy đủ thuộc tính)
cat > main.tf << 'EOF'
resource "aws_instance" "legacy" {
  # Terraform sẽ điền thuộc tính sau khi import
}
EOF

# 2. Import resource
terraform import aws_instance.legacy i-0abc123def456789

# 3. Xem state để biết thuộc tính thực tế
terraform state show aws_instance.legacy

# 4. Cập nhật code HCL với đầy đủ thuộc tính

# 5. Chạy plan — phải hiện "No changes"
terraform plan
```

---

## 9. moved Block — Di Chuyển Trong Code

Thay vì dùng `terraform state mv` (command line, không audit trail), Terraform 1.1+ hỗ trợ `moved` block trong HCL:

```hcl
# moved.tf — Khai báo trong code, có thể review qua PR
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

moved {
  from = aws_instance.app[0]
  to   = module.app_tier.aws_instance.main["primary"]
}
```

**Lợi ích so với `state mv`:**

| Tiêu Chí | `terraform state mv` | `moved` block |
|----------|---------------------|---------------|
| Audit trail | Chỉ trong terminal | Trong Git history |
| Team review | Không | Qua PR review |
| Tự động | Cần chạy thủ công | Terraform tự apply khi plan |
| Idempotent — Chạy nhiều lần an toàn | Không | Có |
| Phù hợp với CI/CD | Khó | Hoàn toàn phù hợp |

**Sau khi apply xong, xóa moved block đi** — nó chỉ cần thiết một lần.

---

## 10. Workflow Thực Tế — Các Kịch Bản Hay Gặp

### Kịch bản 1: Refactor module không destroy resource

```bash
# Hiện tại: resource nằm ở root module
# Mục tiêu: di chuyển vào module "vpc"

# Bước 1: Viết moved block
cat >> moved.tf << 'EOF'
moved {
  from = aws_vpc.main
  to   = module.vpc.aws_vpc.main
}
moved {
  from = aws_subnet.private[0]
  to   = module.vpc.aws_subnet.private[0]
}
EOF

# Bước 2: Cập nhật code (xóa resource ở root, thêm module block)

# Bước 3: Plan để verify
terraform plan   # Phải hiện: "0 to add, 0 to change, 0 to destroy"

# Bước 4: Apply
terraform apply

# Bước 5: Xóa moved block sau khi apply thành công
```

### Kịch bản 2: Đổi từ count sang for_each

```bash
# Backup state
terraform state pull > backup.tfstate

# Di chuyển từng resource
terraform state mv 'aws_instance.app[0]' 'aws_instance.app["web-1"]'
terraform state mv 'aws_instance.app[1]' 'aws_instance.app["web-2"]'

# Cập nhật code từ count sang for_each
# Plan để verify
terraform plan   # "No changes" là thành công
```

### Kịch bản 3: Adopt existing resource — Tiếp quản tài nguyên hiện có

```bash
# Resource S3 bucket đã tồn tại, muốn đưa vào Terraform quản lý

# Bước 1: Viết resource block
# Bước 2: Import
terraform import aws_s3_bucket.data mycompany-data-lake

# Bước 3: Sync code với thực tế
terraform state show aws_s3_bucket.data  # Xem thuộc tính hiện có

# Bước 4: Cập nhật code cho khớp
# Bước 5: Plan phải cho "No changes"
terraform plan
```

---

## 11. Câu Hỏi Phỏng Vấn

### Q: Dùng `terraform state mv` khi nào?

**Trả lời mẫu:**
> Dùng khi cần đổi tên hoặc di chuyển resource trong code mà không muốn destroy và recreate tài nguyên thực tế. Ví dụ phổ biến: refactor module cũ vào module mới, đổi từ count sang for_each, hay tách project lớn thành nhỏ hơn. Với Terraform 1.1+, tôi ưu tiên dùng `moved` block trong HCL vì có thể review qua PR và tự động hóa trong CI/CD.

### Q: `terraform state rm` có xóa tài nguyên trên cloud không?

**Trả lời mẫu:**
> Không. `terraform state rm` chỉ xóa reference — tham chiếu — trong state file, tài nguyên thực tế trên cloud vẫn tồn tại nguyên vẹn. Terraform đơn giản là "quên" rằng nó đang quản lý tài nguyên đó. Lệnh để thực sự xóa tài nguyên là `terraform destroy` hoặc xóa resource block khỏi code rồi apply.

### Q: `terraform import` và `moved` block khác nhau thế nào?

**Trả lời mẫu:**
> `terraform import` dùng để đưa resource đã tồn tại trên cloud vào Terraform state từ bên ngoài — khi Terraform chưa biết về nó. `moved` block dùng để báo Terraform rằng một resource đã được quản lý đang được đổi tên hoặc di chuyển trong code, để Terraform không destroy và recreate lại. `import` là về "bắt đầu quản lý mới", còn `moved` là về "tái tổ chức code không phá hủy hạ tầng".

---

## 🔗 Đọc Tiếp

- [5-state-recovery.md](./5-state-recovery.md) — Phục hồi khi state bị corrupt

---

**Cập Nhật Lần Cuối:** 2026-05-12
