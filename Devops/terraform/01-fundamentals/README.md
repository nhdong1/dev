# 01 — Nền Tảng Terraform (Fundamentals)

> Phần này xây dựng nền tảng vững chắc: IaC là gì, cú pháp HCL, cách Provider và Resource hoạt động, và vòng đời của một lần triển khai hạ tầng.

---

## 📚 Nội Dung Phần Này

| File | Chủ Đề | Thời Gian Đọc |
|------|--------|---------------|
| [1-what-is-iac.md](./1-what-is-iac.md) | IaC — Infrastructure as Code — là gì, lợi ích, so sánh công cụ | 20 phút |
| [2-hcl-syntax.md](./2-hcl-syntax.md) | HCL — HashiCorp Configuration Language — cú pháp đầy đủ | 30 phút |
| [3-providers-resources.md](./3-providers-resources.md) | Provider — Nhà cung cấp & Resource — Tài nguyên cơ bản | 25 phút |
| [4-variables-outputs.md](./4-variables-outputs.md) | Variables — Biến, Locals — Biến cục bộ, Outputs — Đầu ra | 25 phút |
| [5-lifecycle.md](./5-lifecycle.md) | init → plan → apply → destroy — Vòng đời tài nguyên | 20 phút |

**Tổng thời gian học lý thuyết:** ~2 giờ  
**Thời gian thực hành:** 2-4 giờ  
**Tổng:** 4-6 giờ

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Giải thích IaC là gì và tại sao Terraform được ưa chuộng
- [ ] Đọc và viết được HCL — HashiCorp Configuration Language — cơ bản
- [ ] Cấu hình Provider — Nhà cung cấp — và khai báo Resource — Tài nguyên
- [ ] Dùng Variables — Biến, Locals — Biến cục bộ, và Outputs — Đầu ra hiệu quả
- [ ] Hiểu và thực hiện đúng vòng đời: `init` → `plan` → `apply` → `destroy`

---

## 🗺️ Lộ Trình Học Phần Này

```
Tuần 1, Ngày 1-2:
  └── 1-what-is-iac.md        → Nắm bức tranh tổng thể
  └── 2-hcl-syntax.md         → Làm quen cú pháp

Tuần 1, Ngày 3-4:
  └── 3-providers-resources.md → Viết resource đầu tiên
  └── 4-variables-outputs.md  → Tham số hoá configuration

Tuần 1, Ngày 5:
  └── 5-lifecycle.md          → Chạy full workflow
  └── Thực hành lab           → Tạo tài nguyên trên cloud
```

---

## 🔑 Khái Niệm Cốt Lõi Của Phần Này

### IaC — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã

Thay vì click trên console hoặc chạy lệnh thủ công, bạn mô tả hạ tầng bằng code. Code được version-control — quản lý phiên bản, review, và test như application code.

### Declarative vs Imperative — Khai Báo vs Mệnh Lệnh

Terraform dùng cách tiếp cận **declarative** — khai báo:
- Bạn mô tả **trạng thái mong muốn** ("tôi muốn có 3 EC2 instance")
- Terraform tự tìm ra **cách đạt đến** trạng thái đó
- Khác với imperative — mệnh lệnh: "tạo instance 1, tạo instance 2, tạo instance 3"

### State — Trạng Thái

Terraform theo dõi hạ tầng thực tế qua file `terraform.tfstate`. Đây là cầu nối giữa code và reality — thực tế. Hiểu state là hiểu Terraform.

### Provider — Nhà Cung Cấp

Plugin dịch Terraform resource sang API calls — lời gọi API — của cloud provider. AWS provider, GCP provider, Azure provider đều là những plugin riêng biệt.

---

## ⚠️ Các Lỗi Hay Gặp Của Người Mới

| Lỗi | Hậu Quả | Cách Tránh |
|-----|---------|-----------|
| Commit `terraform.tfstate` lên git | Lộ secrets, conflict khi làm nhóm | Thêm vào `.gitignore`, dùng remote backend |
| Chạy `apply` mà không xem `plan` | Xóa nhầm tài nguyên production | Luôn đọc plan output trước khi confirm |
| Hardcode credentials trong `.tf` file | Lộ access key, secret key | Dùng environment variables hoặc IAM role |
| Không dùng version constraints | Provider updates gây breaking change | Pin provider version trong `required_providers` |
| Tạo tất cả resource trong một file | Code khó đọc, khó maintain | Tách theo logic: `main.tf`, `variables.tf`, `outputs.tf` |

---

## 🛠️ Thiết Lập Môi Trường Trước Khi Bắt Đầu

```bash
# 1. Cài Terraform (macOS)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# 1. Cài Terraform (Linux)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# 2. Kiểm tra version
terraform version
# Kết quả mong muốn: Terraform v1.7.x hoặc mới hơn

# 3. Cài công cụ hỗ trợ
brew install tflint      # Linter — kiểm tra cú pháp và best practices
brew install tfenv       # Quản lý nhiều phiên bản Terraform

# 4. Cài extension VS Code
# HashiCorp Terraform (hashicorp.terraform)
# Cung cấp: syntax highlighting, autocomplete, go-to-definition
```

---

## 📝 Bài Tập Thực Hành Phần Này

### Bài 1: Hello World Terraform

```hcl
# main.tf — File cấu hình chính
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Xin chào từ Terraform!\n"
}
```

```bash
terraform init    # Tải provider
terraform plan    # Xem sẽ tạo gì
terraform apply   # Tạo file
cat hello.txt     # Kiểm tra kết quả
terraform destroy # Dọn dẹp
```

### Bài 2: AWS S3 Bucket (Cần AWS account)

```hcl
# provider.tf
provider "aws" {
  region = "ap-southeast-1"  # Singapore
}

# main.tf
resource "aws_s3_bucket" "learning" {
  bucket = "terraform-learning-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}

# outputs.tf
output "bucket_name" {
  value = aws_s3_bucket.learning.bucket
}
```

### Bài 3: Tham Số Hoá Với Variables

Refactor — tái cấu trúc — bài 2 để dùng variables:

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region để triển khai tài nguyên"
  type        = string
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Môi trường: dev, staging, prod"
  type        = string
}
```

---

## 🔗 Liên Kết Điều Hướng

- ← [README tổng quan](../README.md)
- → [1. IaC là gì](./1-what-is-iac.md) — Bước tiếp theo
- → [02 State Management](../02-state-management/README.md) — Sau khi hoàn thành phần này

---

**Cập Nhật Lần Cuối:** 2026-05-12  
**Trạng Thái:** ✅ Hoàn thành
