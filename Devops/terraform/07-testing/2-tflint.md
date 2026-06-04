# 2 — TFLint — Kiểm Tra Lỗi và Best Practices Nâng Cao

> TFLint là linter — công cụ kiểm tra mã tĩnh — cho Terraform, bổ sung những gì `terraform validate` không kiểm tra được: lỗi provider-specific — lỗi đặc thù provider, deprecated syntax — cú pháp đã lỗi thời, và vi phạm best practices.

---

## TFLint Là Gì và Tại Sao Cần?

`terraform validate` chỉ kiểm tra cú pháp HCL hợp lệ. Nó **không** phát hiện:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t4.micro"   # Lỗi! t4.micro không tồn tại trên AWS
                                # validate không biết, TFLint biết
}
```

TFLint dùng **plugin hệ thống** — Plugin system — để hiểu các provider cụ thể như AWS, GCP, Azure và bắt lỗi mà Terraform CLI không thể phát hiện lúc validate.

### Những Gì TFLint Bắt Được

| Loại Lỗi | Ví Dụ | terraform validate | TFLint |
|----------|-------|--------------------|--------|
| Cú pháp HCL sai | Thiếu dấu `}` | ✅ | ✅ |
| Instance type không tồn tại | `t4.micro` | ❌ | ✅ |
| Deprecated argument | `associate_public_ip_address` trong resource cũ | ❌ | ✅ |
| Naming convention | Tên resource chứa dấu `-` thay `_` | ❌ | ✅ (với rule) |
| Unused declarations | Variable khai báo nhưng không dùng | ❌ | ✅ |
| Missing required tags | Resource thiếu tag `Environment` | ❌ | ✅ (với rule tùy chỉnh) |

---

## Cài Đặt

```bash
# macOS
brew install tflint

# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Docker
docker run --rm -v $(pwd):/data -t ghcr.io/terraform-linters/tflint

# Kiểm tra version
tflint --version
```

---

## Cấu Hình — `.tflint.hcl`

```hcl
# .tflint.hcl — Đặt ở root project hoặc thư mục module

plugin "aws" {
  enabled = true
  version = "0.29.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

plugin "google" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-google"
}

plugin "azurerm" {
  enabled = true
  version = "0.26.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}

# Cấu hình rule tích hợp sẵn
rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true

  resource {
    format = "snake_case"   # Buộc dùng snake_case — chữ_thường_gạch_dưới
  }

  variable {
    format = "snake_case"
  }
}

rule "terraform_documented_variables" {
  enabled = true   # Yêu cầu mọi variable có description
}

rule "terraform_documented_outputs" {
  enabled = true   # Yêu cầu mọi output có description
}
```

---

## Cách Chạy TFLint

```bash
# Bước 1: Khởi tạo — tải plugin về
tflint --init

# Bước 2: Chạy lint trong thư mục hiện tại
tflint

# Chạy đệ quy — recursive — tất cả module
tflint --recursive

# Chỉ hiển thị lỗi, bỏ qua warning
tflint --minimum-failure-severity=error

# Output dạng JSON — để tích hợp với tool khác
tflint --format=json

# Chạy với file cấu hình cụ thể
tflint --config=.tflint.hcl
```

### Ví Dụ Output

```
$ tflint

3 issue(s) found:

Warning: aws_instance_invalid_type (aws_instance_invalid_instance_type)

  on main.tf line 5:
   5:   instance_type = "t4.micro"

"t4.micro" is an invalid value as instance_type.
Reference: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance

Error: terraform_required_version (terraform_required_version)

  on main.tf line 1:
   1: terraform {

`required_version` attribute is missing.
Reference: https://www.terraform.io/docs/language/settings/index.html#specifying-a-required-terraform-version

Warning: terraform_documented_variables (terraform_documented_variables)

  on variables.tf line 1:
   1: variable "instance_count" {

`description` attribute is missing.
```

---

## Rules Quan Trọng Theo Nhóm

### Rules Terraform Built-in

```hcl
# Yêu cầu khai báo required_version trong terraform block
rule "terraform_required_version" {
  enabled = true
}

# Yêu cầu khai báo required_providers đầy đủ
rule "terraform_required_providers" {
  enabled = true
}

# Cảnh báo khi dùng data source deprecated
rule "terraform_deprecated_index" {
  enabled = true
}

# Không cho phép biến không dùng
rule "terraform_unused_declarations" {
  enabled = true
}

# Yêu cầu description cho mọi variable
rule "terraform_documented_variables" {
  enabled = true
}

# Yêu cầu description cho mọi output
rule "terraform_documented_outputs" {
  enabled = true
}
```

### Rules AWS Provider

```hcl
plugin "aws" {
  enabled = true
  version = "0.29.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Rule được tự động bật khi dùng plugin AWS:
# - aws_instance_invalid_type         : instance type không tồn tại
# - aws_db_instance_invalid_type      : RDS instance type không tồn tại
# - aws_elasticache_cluster_invalid_type : cache node type không tồn tại
# - aws_*_invalid_*                   : nhiều validation khác

# Ví dụ lỗi rule bắt được:
# aws_instance: "t4g.micro" vs "t4g.micro" → t4.micro không hợp lệ
# aws_db_instance: "db.t5.micro" → db.t5 không tồn tại
```

---

## Tích Hợp Với CI/CD

### GitHub Actions

```yaml
name: Terraform Lint

on:
  pull_request:
    paths:
      - '**.tf'

jobs:
  tflint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: v0.50.0

      - name: Init TFLint (tải plugin)
        run: tflint --init
        env:
          GITHUB_TOKEN: ${{ github.token }}   # Cần để tải plugin từ GitHub

      - name: Run TFLint
        run: tflint --recursive --format=compact

      - name: Upload TFLint Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: tflint-results
          path: tflint-report.json
```

### GitLab CI

```yaml
tflint:
  stage: validate
  image: ghcr.io/terraform-linters/tflint:latest
  script:
    - tflint --init
    - tflint --recursive
  only:
    changes:
      - "**/*.tf"
      - "**/*.tfvars"
```

---

## Custom Rules — Luật Tùy Chỉnh

Khi cần kiểm tra quy tắc riêng của tổ chức, có thể viết plugin TFLint tùy chỉnh bằng Go, hoặc dùng OPA — Open Policy Agent — (xem [4-checkov-opa.md](4-checkov-opa.md)).

Ví dụ rule custom bằng cách dùng `terraform_naming_convention`:

```hcl
# .tflint.hcl — Quy ước đặt tên của tổ chức
rule "terraform_naming_convention" {
  enabled = true

  # Tất cả resource phải theo pattern: <env>_<service>_<role>
  # Ví dụ: prod_web_alb, dev_db_primary
  resource {
    format = "snake_case"
  }

  # Module call phải dùng snake_case
  module {
    format = "snake_case"
  }

  # Variable phải dùng snake_case
  variable {
    format = "snake_case"
  }

  # Data source phải dùng snake_case
  data {
    format = "snake_case"
  }
}
```

---

## Bỏ Qua Lỗi TFLint — Ignore Rules

```hcl
# Bỏ qua rule cụ thể cho một resource (dùng annotation)
resource "aws_instance" "legacy" {
  # tflint-ignore: aws_instance_invalid_type
  instance_type = "t1.micro"   # Instance cũ, vẫn tồn tại nhưng TFLint cảnh báo
  ami           = "ami-12345678"
}
```

Hoặc trong `.tflint.hcl`:

```hcl
# Tắt hoàn toàn một rule (không khuyến khích)
rule "aws_instance_invalid_type" {
  enabled = false
}
```

---

## So Sánh TFLint vs Checkov vs tfsec

| Tiêu Chí | TFLint | Checkov | tfsec |
|---------|--------|---------|-------|
| **Mục đích chính** | Linting, correctness | Security & compliance | Security |
| **Loại lỗi** | Provider validation, naming, best practices | Misconfiguration, CIS benchmarks | Security vulnerabilities |
| **Tốc độ** | Rất nhanh | Nhanh | Nhanh |
| **Plugin system** | ✅ (AWS, GCP, Azure) | ✅ (nhiều provider) | ✅ |
| **Custom rules** | Cần viết Go plugin | Python / YAML | Rego (OPA) |
| **Chạy khi nào** | Pre-commit, CI | CI, CD | CI, CD |

**Thực tế:** Dùng cả 3 — TFLint cho correctness, Checkov hoặc tfsec cho security.

---

## Câu Hỏi Phỏng Vấn

**Q: TFLint khác `terraform validate` ở điểm nào?**

A: `terraform validate` kiểm tra cú pháp HCL và tham chiếu nội bộ, nhưng không biết về các giá trị hợp lệ của provider — ví dụ instance type AWS nào tồn tại. TFLint có plugin cho từng provider, giúp phát hiện lỗi runtime tiềm ẩn như dùng instance type không tồn tại, deprecated arguments, hoặc vi phạm naming convention trước khi apply.

**Q: Khi nào nên chạy TFLint trong pipeline?**

A: TFLint nên chạy ở giai đoạn sớm nhất — thường sau `terraform fmt -check` và `terraform validate`. Chạy mỗi khi có Pull Request thay đổi file `.tf`. Có thể cấu hình pre-commit hook để bắt lỗi ngay trên máy developer trước khi push.

---

**Tiếp theo:** [3-terratest.md](3-terratest.md) — Terratest — Unit & Integration Testing thực tế trên cloud
