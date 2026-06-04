# 📦 03 — Modules — Mô-đun Terraform

> Module — Mô-đun — là đơn vị đóng gói và tái sử dụng cơ bản trong Terraform.
> Thiết kế module tốt giúp hạ tầng trở nên dễ đọc, dễ kiểm thử và dễ bảo trì.

---

## 🗺️ Mục Lục Phần Này

| File                         | Nội Dung                                              | Thời Gian |
| ---------------------------- | ----------------------------------------------------- | --------- |
| `1-module-structure.md`      | Cấu trúc thư mục module chuẩn, file bắt buộc          | 45 phút   |
| `2-input-output.md`          | Variables, Outputs, Type Constraints — Ràng buộc kiểu | 60 phút   |
| `3-module-registry.md`       | Terraform Registry public & private                   | 45 phút   |
| `4-module-versioning.md`     | Semantic Versioning — Đánh số phiên bản — và pinning  | 30 phút   |
| `5-composition-patterns.md`  | Flat, Nested, Wrapper — Các kiểu ghép module          | 60 phút   |

**Tổng thời gian:** ~4–5 giờ (bao gồm thực hành)

---

## 🎯 Tại Sao Module Quan Trọng

### Vấn Đề Khi Không Dùng Module

```
Không có module:
  ├── main.tf          (800 dòng HCL lộn xộn)
  ├── variables.tf     (60 biến không có tổ chức)
  └── outputs.tf       (40 output khó hiểu)

Hậu quả:
  - Copy/paste code giữa môi trường → không nhất quán
  - Một thay đổi phải sửa nhiều chỗ → dễ sót
  - Không ai hiểu code của người khác viết
  - Không thể kiểm thử độc lập từng thành phần
```

### Lợi Ích Khi Dùng Module

```
Có module:
  ├── modules/
  │   ├── vpc/          (Mạng ảo — VPC)
  │   ├── eks-cluster/  (Kubernetes cluster)
  │   └── rds/          (Cơ sở dữ liệu quan hệ)
  └── environments/
      ├── dev/          (Gọi modules, cấu hình dev)
      └── prod/         (Gọi modules, cấu hình prod)

Lợi ích:
  ✅ Viết một lần, dùng nhiều môi trường
  ✅ Thay đổi một chỗ, áp dụng toàn bộ
  ✅ Dễ kiểm thử từng module độc lập
  ✅ Đóng gói phức tạp, expose interface đơn giản
```

---

## 🧱 Khái Niệm Cốt Lõi

### Module Là Gì?

```
Module = một thư mục chứa file .tf

Mọi cấu hình Terraform đều là module:
  - Root module — Module gốc: thư mục bạn chạy terraform apply
  - Child module — Module con: thư mục được gọi bằng block "module {}"
  - Published module — Module công bố: module trên Terraform Registry
```

### Cách Gọi Module

```hcl
# Root module gọi child module
module "vpc" {
  source  = "./modules/vpc"        # local path — đường dẫn cục bộ
  version = "~> 3.0"               # chỉ áp dụng cho registry

  # Input variables — Biến đầu vào
  vpc_cidr   = "10.0.0.0/16"
  region     = var.aws_region
  env_name   = var.environment
}

# Dùng output của module
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
}
```

---

## 📐 Nguyên Tắc Thiết Kế Module Tốt

### 1. Single Responsibility — Trách Nhiệm Đơn

```
❌ Module làm quá nhiều việc:
   module "everything" — tạo VPC + EKS + RDS + IAM + monitoring

✅ Module có phạm vi rõ ràng:
   module "vpc"         — chỉ tạo mạng
   module "eks-cluster" — chỉ tạo Kubernetes cluster
   module "rds"         — chỉ tạo database
```

### 2. Explicit Interface — Interface Tường Minh

```hcl
# ✅ Variables rõ ràng, có description và validation
variable "environment" {
  description = "Môi trường triển khai (dev/staging/prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Chỉ chấp nhận: dev, staging, prod."
  }
}
```

### 3. No Hard-coded Values — Không Giá Trị Cứng

```hcl
# ❌ Giá trị cứng — hard-coded
resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
}

# ✅ Nhận qua variables
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

### 4. Sensible Defaults — Giá Trị Mặc Định Hợp Lý

```hcl
# ✅ Default an toàn, có thể override
variable "enable_deletion_protection" {
  description = "Bật bảo vệ xoá tình cờ"
  type        = bool
  default     = true   # Mặc định an toàn cho production
}
```

### 5. Backward Compatibility — Tương Thích Ngược

```
Khi cập nhật module:
  - Thêm variable mới: OK nếu có default
  - Đổi tên variable: BREAKING — cần moved block
  - Xoá variable: BREAKING — kiểm tra tất cả callers trước
  - Thêm resource mới: Có thể gây replace nếu tên thay đổi
```

---

## 🔄 Vòng Đời Phát Triển Module

```
1. Thiết kế interface     → Xác định variables và outputs cần thiết
         ↓
2. Viết code              → Tạo resources theo module structure
         ↓
3. Viết tests             → Dùng Terratest hoặc terraform test
         ↓
4. Version & tag          → Git tag: v1.0.0
         ↓
5. Publish (nếu cần)      → Terraform Registry hoặc private registry
         ↓
6. Dùng trong projects    → source = "module-source" + version pin
         ↓
7. Iterate & maintain     → Cập nhật, giữ backward compatibility
```

---

## ⚠️ Anti-Patterns — Kiểu Sai Cần Tránh

| Anti-Pattern                     | Hậu Quả                                | Giải Pháp                        |
| -------------------------------- | -------------------------------------- | -------------------------------- |
| Module quá lớn (God Module)      | Khó tái sử dụng, khó test             | Chia thành modules nhỏ hơn       |
| Không pin version                | Cập nhật ngoài ý muốn, phá hỏng infra  | Luôn dùng version constraints    |
| Expose quá nhiều variables       | Interface phức tạp, khó dùng           | Đóng gói, chỉ expose cần thiết   |
| Nested modules quá sâu           | Khó debug, khó hiểu dependencies       | Tối đa 2-3 tầng nesting          |
| Hardcode provider config trong module | Module không tái sử dụng được   | Dùng provider aliases            |
| Không có outputs                 | Caller không lấy được thông tin        | Expose tất cả thông tin hữu ích  |

---

## 🗂️ Phân Loại Module Theo Mục Đích

### Infrastructure Modules — Module Hạ Tầng Cơ Sở

```
Xây dựng tài nguyên low-level:
  - modules/networking/vpc
  - modules/networking/security-groups
  - modules/compute/ec2-autoscaling
  - modules/database/rds-postgres
```

### Service Modules — Module Dịch Vụ

```
Kết hợp nhiều infrastructure modules thành dịch vụ hoàn chỉnh:
  - modules/services/backend-api    (EC2 + ALB + RDS + IAM)
  - modules/services/data-pipeline  (Kinesis + Lambda + S3)
  - modules/services/kubernetes-app (EKS + ECR + Route53)
```

### Environment Modules — Module Môi Trường

```
Cấu hình hoàn chỉnh cho một môi trường:
  - environments/dev/
  - environments/staging/
  - environments/prod/
  (Đây là root module — chỉ gọi service modules, không tạo resource trực tiếp)
```

---

## 📋 Checklist Module Production-Ready

```
Interface:
  [ ] Tất cả variables có description rõ ràng
  [ ] Variables quan trọng có validation
  [ ] Outputs đủ để caller dùng tiếp
  [ ] Không có hard-coded values

Code Quality:
  [ ] Chạy được terraform validate
  [ ] Chạy được terraform fmt
  [ ] Không có circular dependencies
  [ ] Resource naming nhất quán

Documentation:
  [ ] README.md với usage example
  [ ] Variables.tf đủ description
  [ ] Outputs.tf đủ description
  [ ] CHANGELOG.md cho versions

Testing:
  [ ] Có ít nhất một test cơ bản
  [ ] Test được các edge cases quan trọng

Versioning:
  [ ] Có Git tag với semantic version
  [ ] CHANGELOG ghi rõ breaking changes
```

---

## 🚀 Bước Tiếp Theo

1. **[1-module-structure.md](1-module-structure.md)** — Cấu trúc thư mục chuẩn, file nào cần thiết
2. **[2-input-output.md](2-input-output.md)** — Design variables và outputs hiệu quả
3. **[3-module-registry.md](3-module-registry.md)** — Dùng và publish module lên registry
4. **[4-module-versioning.md](4-module-versioning.md)** — Version management và upgrade strategy
5. **[5-composition-patterns.md](5-composition-patterns.md)** — Kiến trúc ghép modules

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Trạng Thái:** ✅ Hoàn thành
