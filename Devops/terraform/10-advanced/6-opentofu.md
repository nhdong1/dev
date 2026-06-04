# OpenTofu — Nhánh Open-source của Terraform

> **OpenTofu** là fork — nhánh tách — open-source của Terraform, được Linux Foundation quản lý, ra đời sau khi HashiCorp thay đổi license Terraform từ MPL-2.0 sang BSL-1.1 vào tháng 8/2023. OpenTofu cam kết duy trì license thực sự open-source (MPL-2.0) và tương thích với Terraform.

---

## 1. Bối Cảnh Lịch Sử

### Timeline — Dòng Thời Gian Sự Kiện

```
Tháng 8/2023:
  HashiCorp thông báo chuyển Terraform từ MPL-2.0 → BSL-1.1
  (Business Source License — License Doanh Nghiệp)
  BSL-1.1 cấm dùng trong sản phẩm cạnh tranh trực tiếp với HashiCorp

Tháng 9/2023:
  OpenTF Manifesto được công bố
  Cộng đồng (Spacelift, Gruntwork, Env0, Scalr...) ký tên phản đối

Tháng 9/2023:
  OpenTF fork được tạo từ Terraform v1.5.7 (bản cuối MPL-2.0)

Tháng 1/2024:
  OpenTofu v1.6.0 ra mắt — phiên bản stable đầu tiên
  Linux Foundation tiếp nhận quản lý dự án

Tháng 2/2025:
  IBM mua lại HashiCorp → tương lai Terraform trở nên không chắc chắn

2025+:
  OpenTofu tiếp tục phát triển độc lập
  Thêm tính năng mới không có trong Terraform
```

### BSL vs MPL — Khác Nhau Gì?

| | **MPL-2.0** (Mozilla Public License) | **BSL-1.1** (Business Source License) |
|---|--------------------------------------|---------------------------------------|
| **Open source** | ✅ Thực sự open source | ❌ Source available, không phải OSI open source |
| **Dùng thương mại** | ✅ Được phép | ⚠️ Cấm nếu cạnh tranh với HashiCorp |
| **Modify & distribute** | ✅ Được phép | ⚠️ Hạn chế |
| **SaaS với Terraform** | ✅ Được phép | ❌ Cấm nếu là "competitive offering" |
| **Tự dùng nội bộ** | ✅ | ✅ Vẫn cho phép |

---

## 2. OpenTofu vs Terraform — So Sánh

### Tương Thích — Compatibility

```
OpenTofu 1.6.x ≈ Terraform 1.6.x (gần như tương đương)
OpenTofu 1.7.x  > Terraform 1.7.x (có tính năng mới)
OpenTofu 1.8.x  > Terraform 1.8.x (thêm nhiều tính năng)
```

**Tương thích ở mức:**
- ✅ HCL syntax — cú pháp giống nhau
- ✅ State file format — cùng format
- ✅ Provider compatibility — dùng cùng provider
- ✅ Module compatibility — dùng được module Terraform Registry
- ✅ Backend compatibility — cùng backend (S3, GCS, Azure Blob...)

**Không tương thích:**
- ❌ Tính năng mới của OpenTofu chưa có trong Terraform
- ❌ Một số tính năng mới của Terraform chưa được port sang OpenTofu

### Tính Năng OpenTofu Không Có Trong Terraform

```
OpenTofu 1.7+:
  ✨ State encryption — Mã hóa state file tại chỗ (không cần KMS bên ngoài)
  ✨ Provider-defined functions — Provider có thể định nghĩa function HCL
  ✨ Loopable import blocks — Import nhiều resource với for_each

OpenTofu 1.8+:
  ✨ Variable/locals trong backend và provider config
  ✨ Improved error messages — Thông báo lỗi chi tiết hơn
  ✨ OCI registry support — Hỗ trợ OCI registry cho module và provider
```

---

## 3. State Encryption — Mã Hóa State File

Tính năng độc đáo của OpenTofu, không có trong Terraform.

### Tại Sao Cần State Encryption?

Terraform state file chứa thông tin nhạy cảm: database passwords, private keys, API tokens... dưới dạng plaintext. Dù lưu trong S3 với SSE — Server Side Encryption — dữ liệu vẫn đọc được bởi bất kỳ ai có quyền S3.

### Cấu Hình State Encryption

```hcl
# OpenTofu encryption configuration
terraform {
  required_version = ">= 1.7"

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }

  # State encryption — Mã hóa state
  encryption {
    # Dùng AWS KMS — Key Management Service — để quản lý key mã hóa
    key_provider "aws_kms" "main" {
      kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd1234-..."
      region     = "us-east-1"
    }

    # Phương pháp mã hóa AES-GCM
    method "aes_gcm" "main" {
      keys = key_provider.aws_kms.main
    }

    # Áp dụng mã hóa cho state
    state {
      method = method.aes_gcm.main

      # Fallback khi decrypt: thử mã hóa trước, nếu fail thì đọc plaintext
      # Dùng khi migrate từ unencrypted → encrypted
      fallback {
        method = method.unencrypted
      }
    }

    # Áp dụng mã hóa cho plan output
    plan {
      method = method.aes_gcm.main
    }
  }
}
```

### State Encryption Với Passphrase

```hcl
terraform {
  encryption {
    # Dùng passphrase đơn giản (cho dev/test)
    key_provider "pbkdf2" "main" {
      passphrase = var.state_encryption_password
    }

    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }

    state {
      method = method.aes_gcm.main
    }
  }
}
```

---

## 4. Migration — Chuyển Đổi Từ Terraform Sang OpenTofu

### Quy Trình Migration An Toàn

```bash
# Bước 1: Kiểm tra phiên bản tương thích
# OpenTofu cần >= Terraform version đang dùng
terraform version  # Ví dụ: 1.6.6

# Bước 2: Cài OpenTofu
# macOS
brew install opentofu

# Linux
curl -Lo /tmp/tofu.tar.gz https://github.com/opentofu/opentofu/releases/download/v1.8.0/tofu_1.8.0_linux_amd64.tar.gz
tar -xzf /tmp/tofu.tar.gz -C /tmp/
sudo mv /tmp/tofu /usr/local/bin/

# Windows (PowerShell)
winget install OpenTofu.OpenTofu

# Bước 3: Kiểm tra
tofu version
# OpenTofu v1.8.0
# on linux_amd64

# Bước 4: Backup state trước khi migrate
terraform state pull > terraform.tfstate.backup

# Bước 5: Chạy tofu init trong thư mục project
# OpenTofu sẽ đọc cùng terraform.lock.hcl và state
tofu init

# Bước 6: Verify plan giống nhau
tofu plan  # Phải tương đương terraform plan

# Bước 7: Nếu plan OK, thay terraform bằng tofu trong tất cả script
```

### Thay Đổi Trong `.terraform.lock.hcl`

```hcl
# Terraform lock file — OpenTofu đọc được file này
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:...",
  ]
}
```

OpenTofu dùng cùng format lock file, chỉ thêm prefix `tofu` vào binary.

### CI/CD Migration

```yaml
# GitHub Actions — Before (Terraform)
- name: Setup Terraform
  uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: "1.6.6"

- name: Terraform Plan
  run: terraform plan

# After (OpenTofu)
- name: Setup OpenTofu
  uses: opentofu/setup-opentofu@v1
  with:
    tofu_version: "1.8.0"

- name: OpenTofu Plan
  run: tofu plan
```

---

## 5. Provider-defined Functions — Hàm Định Nghĩa Bởi Provider

Tính năng mới của OpenTofu 1.7+ — provider có thể expose functions cho HCL:

```hcl
# Ví dụ: AWS provider expose function để xử lý ARN
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Dùng provider-defined function
locals {
  # provider::aws::arn_parse — parse ARN thành components
  bucket_arn    = "arn:aws:s3:::my-bucket"
  arn_components = provider::aws::arn_parse(local.bucket_arn)
  # → { partition = "aws", service = "s3", region = "", account = "", resource = "my-bucket" }
}

output "s3_bucket_name" {
  value = local.arn_components.resource
}
```

---

## 6. Loopable Import Blocks — Import Nhiều Resource

```hcl
# OpenTofu 1.7+ — Import nhiều resource cùng lúc với for_each
import {
  for_each = {
    "alice" = "arn:aws:iam::123456789012:user/alice"
    "bob"   = "arn:aws:iam::123456789012:user/bob"
    "carol" = "arn:aws:iam::123456789012:user/carol"
  }

  id = each.value
  to = aws_iam_user.team[each.key]
}

resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}
```

Terraform 1.x chỉ cho phép import một resource mỗi import block — phải viết nhiều block cho nhiều resource.

---

## 7. So Sánh Ecosystem

### Community & Governance — Cộng Đồng & Quản Trị

| | **Terraform** | **OpenTofu** |
|---|--------------|-------------|
| **Tổ chức quản lý** | HashiCorp (IBM) | Linux Foundation |
| **License** | BSL-1.1 | MPL-2.0 |
| **Governance** | Corporate | Open governance (Technical Steering Committee) |
| **Contributor** | HashiCorp internal | Community open |
| **RFC process** | Internal | Public RFC — Request for Comments |

### Tool Support

| Tool | Terraform | OpenTofu |
|------|-----------|---------|
| Atlantis | ✅ | ✅ (v0.26+) |
| Terragrunt | ✅ | ✅ |
| TFLint | ✅ | ✅ |
| tfsec | ✅ | ✅ (tên: tofsec cho OpenTofu-specific features) |
| Checkov | ✅ | ✅ |
| Infracost | ✅ | ✅ |
| Spacelift | ✅ | ✅ |
| env0 | ✅ | ✅ |
| Scalr | ✅ | ✅ |
| Terraform Cloud | ✅ | ❌ (dùng Spacelift/env0 thay thế) |

---

## 8. Quyết Định: Terraform hay OpenTofu?

```
Chọn Terraform nếu:
  □ Đang dùng Terraform Cloud hoặc HCP Terraform
  □ Team quen thuộc và không muốn thay đổi
  □ Không bị ảnh hưởng bởi BSL license
  □ Cần support chính thức từ HashiCorp/IBM

Chọn OpenTofu nếu:
  □ License BSL-1.1 là vấn đề (startup, managed service provider)
  □ Muốn state encryption built-in
  □ Muốn tham gia cộng đồng open governance
  □ Đang dùng Spacelift, env0, Scalr (đã support OpenTofu)
  □ Muốn tính năng mới nhanh hơn (provider functions, loopable import)

Trong cả hai trường hợp:
  □ State file tương thích → có thể switch về sau
  □ Provider tương thích → không cần thay đổi provider code
  □ Module tương thích → Registry module vẫn dùng được
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Tại sao OpenTofu ra đời và nó khác Terraform ở điểm gì?**

A: HashiCorp thay đổi license Terraform từ MPL-2.0 open source sang BSL-1.1 vào 8/2023. BSL cấm dùng Terraform trong sản phẩm cạnh tranh với HashiCorp. Cộng đồng fork Terraform thành OpenTofu dưới Linux Foundation với license MPL-2.0. OpenTofu tương thích gần như hoàn toàn với Terraform nhưng có thêm tính năng mới như state encryption, provider-defined functions, và loopable import blocks.

**Q: Migrate từ Terraform sang OpenTofu có rủi ro gì?**

A: Rủi ro thấp vì state file và provider tương thích. Bước chính là thay binary `terraform` thành `tofu` và cập nhật CI/CD scripts. Cần test kỹ trên môi trường non-production trước. Một số tính năng mới của Terraform sau 1.5.7 có thể chưa có trong OpenTofu và ngược lại.

**Q: State encryption của OpenTofu hoạt động như thế nào?**

A: OpenTofu 1.7+ hỗ trợ mã hóa state file tại client-side trước khi ghi lên backend. Dùng block `encryption {}` trong terraform config, chọn key provider (AWS KMS, GCP KMS, hoặc passphrase PBKDF2) và method (AES-GCM). State được mã hóa trước khi ghi lên S3/GCS, chỉ client có key mới đọc được.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [5-terraform-cdk.md](5-terraform-cdk.md) |
| → Bài tiếp | [7-multi-region-account.md](7-multi-region-account.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐ Nâng Cao
