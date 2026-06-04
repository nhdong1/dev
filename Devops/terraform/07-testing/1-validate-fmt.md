# 1 — terraform validate & terraform fmt — Kiểm Tra Cú Pháp & Định Dạng

> `terraform validate` và `terraform fmt` là hai lệnh đơn giản nhất nhưng bắt buộc phải chạy trước bất kỳ thao tác nào. Chúng bắt lỗi cú pháp và đảm bảo code nhất quán trong team.

---

## terraform validate — Kiểm Tra Tính Hợp Lệ

### Validate Làm Gì?

`terraform validate` kiểm tra:

- **Cú pháp HCL** — HashiCorp Configuration Language — có đúng không
- **Tham chiếu nội bộ** — Biến, resource, module được dùng có tồn tại không
- **Kiểu dữ liệu** — Type mismatch — Sai kiểu giữa input và output
- **Required attributes** — Thuộc tính bắt buộc — có đủ không

`terraform validate` **KHÔNG** kiểm tra:

- Thông tin xác thực cloud — AWS credentials, GCP service account — có hợp lệ không
- Resource có thể tạo được thực tế không
- Giá trị biến runtime — chỉ kiểm tra cấu trúc

### Cách Sử Dụng

```bash
# Kiểm tra thư mục hiện tại (phải chạy terraform init trước)
terraform init -backend=false   # Init không cần backend thực
terraform validate

# Output khi thành công
# Success! The configuration is valid.

# Output khi thất bại
# ╷
# │ Error: Reference to undeclared resource
# │
# │   on main.tf line 12, in resource "aws_instance" "web":
# │   12:   subnet_id = aws_subnet.nonexistent.id
# │
# │ A managed resource "aws_subnet" "nonexistent" has not been declared
# │ in the root module.
# ╵
```

### Validate Trong CI/CD

```yaml
# .github/workflows/terraform.yml
- name: Terraform Init (no backend)
  run: terraform init -backend=false

- name: Terraform Validate
  run: terraform validate
```

### Validate Với -json Flag — Xuất JSON

```bash
terraform validate -json
# Output:
# {
#   "valid": true,
#   "error_count": 0,
#   "warning_count": 0,
#   "diagnostics": []
# }
```

Dùng output JSON để tích hợp với công cụ khác hoặc tạo báo cáo.

### Ví Dụ Lỗi Phổ Biến Validate Bắt Được

```hcl
# Lỗi 1: Tham chiếu sai resource type
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  subnet_id     = aws_vpc.main.id   # Sai! aws_vpc không có thuộc tính để gán subnet_id trực tiếp
}

# Lỗi 2: Kiểu sai
variable "port" {
  type = number
}
resource "aws_security_group_rule" "http" {
  from_port = var.port
  to_port   = "80"   # Lỗi! "80" là string, from_port là number, inconsistent
}

# Lỗi 3: Thiếu required argument
resource "aws_s3_bucket" "data" {
  # Lỗi! bucket name là required trong một số provider version
}
```

---

## terraform fmt — Định Dạng Code Tự Động

### fmt Làm Gì?

`terraform fmt` tự động định dạng file `.tf` và `.tfvars` theo chuẩn HashiCorp:

- Căn chỉnh dấu `=` trong các block
- Thụt lề — Indentation — đúng 2 dấu cách
- Khoảng trắng — Whitespace — nhất quán
- Sắp xếp argument trong một số trường hợp

### Cách Sử Dụng

```bash
# Định dạng tất cả file trong thư mục hiện tại (thay đổi file tại chỗ)
terraform fmt

# Định dạng đệ quy — recursive — tất cả thư mục con
terraform fmt -recursive

# Chỉ kiểm tra xem có cần định dạng không (không thay đổi file)
terraform fmt -check
# Exit code 0 = đã đúng định dạng
# Exit code 3 = có file cần định dạng

# Xem diff — sự khác biệt — trước khi áp dụng
terraform fmt -diff

# Kết hợp check + diff để xem file nào cần sửa
terraform fmt -check -diff -recursive
```

### Trước và Sau fmt

```hcl
# TRƯỚC khi fmt — Không nhất quán
resource "aws_instance" "web" {
  ami="ami-12345678"
  instance_type = "t3.micro"
    tags = {
        Name = "web-server"
    Environment  =   "production"
    }
}

# SAU khi fmt — Nhất quán
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

### fmt Trong Pre-commit Hook — Hook Trước Commit

```bash
# .git/hooks/pre-commit (hoặc dùng pre-commit framework)
#!/bin/sh
terraform fmt -recursive -check
if [ $? -ne 0 ]; then
  echo "❌ Terraform files are not formatted. Run: terraform fmt -recursive"
  exit 1
fi
```

### Dùng pre-commit Framework — Framework Pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: checkov
        args:
          - --args=--quiet
```

Cài đặt và chạy:

```bash
pip install pre-commit
pre-commit install      # Cài hook vào .git/hooks/
pre-commit run --all-files  # Chạy thủ công lần đầu
```

---

## Kết Hợp validate + fmt Trong Workflow

### Local Development — Phát Triển Cục Bộ

```bash
# Trước khi push code
terraform fmt -recursive && terraform validate && echo "✅ All checks passed"
```

### GitHub Actions

```yaml
name: Terraform Checks

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        # Fail nếu có file chưa được định dạng

      - name: Terraform Init
        run: terraform init -backend=false

      - name: Terraform Validate
        run: terraform validate -json | tee validate-output.json
```

### Makefile — File Tự Động Hóa

```makefile
.PHONY: fmt validate check

fmt:
	terraform fmt -recursive

validate:
	terraform init -backend=false
	terraform validate

check: fmt validate
	@echo "✅ All checks passed"

# Dùng: make check
```

---

## Hiểu Exit Codes — Mã Thoát

| Lệnh | Exit Code | Ý Nghĩa |
|------|-----------|---------|
| `terraform validate` | 0 | Hợp lệ |
| `terraform validate` | 1 | Lỗi cú pháp |
| `terraform fmt -check` | 0 | Đã đúng định dạng |
| `terraform fmt -check` | 3 | Cần định dạng lại |

---

## Câu Hỏi Phỏng Vấn

**Q: Khác biệt giữa `terraform validate` và `terraform plan`?**

A: `validate` chỉ kiểm tra cú pháp và tham chiếu trong file cấu hình — không cần credentials, không kết nối cloud. `plan` cần kết nối provider thực, đọc state hiện tại, và tính toán thay đổi cụ thể. `validate` chạy trong < 1 giây, `plan` có thể mất vài phút.

**Q: Tại sao cần `terraform fmt` nếu code chạy được?**

A: Nhất quán về định dạng giúp code review — Xem xét code — dễ hơn, giảm nhiễu trong diff khi nhiều người cùng làm việc. Giống như `gofmt` trong Go hay `black` trong Python — không ảnh hưởng hành vi nhưng cải thiện khả năng đọc và cộng tác.

**Q: `-backend=false` trong `terraform init` nghĩa là gì?**

A: Khởi tạo Terraform mà không kết nối đến remote backend — Nơi lưu state từ xa. Hữu ích trong CI/CD khi chỉ muốn validate cú pháp mà không cần credentials cho S3/GCS. Đặc biệt quan trọng trong các pipeline chạy trên Pull Request từ fork bên ngoài.

---

## Tóm Tắt

| Lệnh | Khi Nào Dùng | Tốc Độ | Cần Credentials? |
|------|-------------|--------|-----------------|
| `terraform fmt` | Trước mỗi commit | < 1 giây | Không |
| `terraform fmt -check` | Trong CI/CD pipeline | < 1 giây | Không |
| `terraform validate` | Sau `fmt`, trước `plan` | < 5 giây | Không (với `-backend=false`) |

Hai lệnh này là **cổng vào đầu tiên** — first gate — trong mọi CI/CD pipeline Terraform. Không tốn chi phí, không rủi ro, nhưng bắt được nhiều lỗi sớm.

---

**Tiếp theo:** [2-tflint.md](2-tflint.md) — TFLint — Kiểm tra lỗi và best practices nâng cao
