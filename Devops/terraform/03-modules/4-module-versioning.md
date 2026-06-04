# 4 — Module Versioning — Quản Lý Phiên Bản Module

> Module versioning — Quản lý phiên bản module — là thực hành đánh số phiên bản
> để kiểm soát thay đổi, tránh breaking changes ngoài ý muốn, và cho phép nhiều
> projects dùng các phiên bản khác nhau của cùng một module.

---

## 🔢 Semantic Versioning — Đánh Số Phiên Bản Ngữ Nghĩa

### Định Nghĩa

```
v MAJOR . MINOR . PATCH
  │        │       └── Sửa lỗi backward-compatible (bug fixes)
  │        └────────── Tính năng mới backward-compatible
  └─────────────────── Thay đổi phá vỡ backward compatibility

Ví dụ: v2.1.3
  - MAJOR = 2: đã có 2 lần breaking change lớn
  - MINOR = 1: sau breaking change lớn nhất, thêm 1 tính năng
  - PATCH = 3: sau lần thêm tính năng, sửa 3 lỗi
```

### Ví Dụ Thực Tế Với Terraform Module

```
v1.0.0 → v1.0.1   PATCH: Sửa bug typo trong tag name
v1.0.1 → v1.1.0   MINOR: Thêm optional variable enable_flow_logs
v1.1.0 → v1.2.0   MINOR: Thêm support cho IPv6 (optional)
v1.2.0 → v2.0.0   MAJOR: Đổi tên variable "vpc_cidr" → "cidr_block" (BREAKING)
v2.0.0 → v2.0.1   PATCH: Fix lỗi trong NAT Gateway resource
```

---

## 🏷️ Git Tags — Gắn Nhãn Phiên Bản

### Tạo Git Tags Cho Module

```bash
# Tạo annotated tag — tag có metadata (khuyến nghị)
git tag -a v1.0.0 -m "Release v1.0.0: Initial stable release"
git push origin v1.0.0

# Tạo tag cho patch version
git tag -a v1.0.1 -m "Fix: typo in resource name tag"
git push origin v1.0.1

# Tạo tag cho minor version
git tag -a v1.1.0 -m "Feat: add optional VPC Flow Logs support"
git push origin v1.1.0

# Tạo tag cho major version (breaking change)
git tag -a v2.0.0 -m "BREAKING: rename vpc_cidr to cidr_block"
git push origin v2.0.0

# Xem tất cả tags
git tag --list "v*" --sort=-version:refname
```

### Quy Trình Release Module

```bash
# 1. Hoàn thành code changes
git add .
git commit -m "feat: add support for multiple NAT gateways"

# 2. Update CHANGELOG.md
# ... thêm entry cho v1.2.0

# 3. Commit CHANGELOG
git add CHANGELOG.md
git commit -m "docs: update CHANGELOG for v1.2.0"

# 4. Tạo tag
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin main
git push origin v1.2.0

# 5. (Nếu dùng Terraform Registry)
#    Registry tự động detect tag mới và publish
```

---

## 📌 Version Constraints — Ràng Buộc Phiên Bản

### Các Ký Hiệu Version Constraint

```hcl
# = chính xác (ít dùng, quá cứng nhắc)
version = "= 1.2.3"

# != loại trừ version cụ thể
version = "!= 1.2.5"   # Tránh version có bug nghiêm trọng

# > < >= <= so sánh
version = ">= 1.2.0"
version = ">= 1.2.0, < 2.0.0"

# ~> pessimistic constraint (phổ biến nhất)
version = "~> 1.2"     # Tương đương >= 1.2, < 2.0
version = "~> 1.2.3"   # Tương đương >= 1.2.3, < 1.3.0
```

### Chiến Lược Pin Version Theo Môi Trường

```hcl
# ===== Module nội bộ (shared modules trong cùng repo) =====
# Dùng local path — không cần version
module "vpc" {
  source = "../../modules/networking/vpc"
}


# ===== Development / Staging =====
# Cho phép minor updates để test tính năng mới
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"    # Tự động lên 20.x, không lên 21.x
}


# ===== Production =====
# Pin exact version — đảm bảo không có gì thay đổi ngoài ý muốn
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.1"     # Chính xác, không có surprises
}


# ===== Git-based modules =====
# Pin theo tag (không bao giờ dùng branch cho production)
module "vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//networking/vpc?ref=v2.1.0"
}

# Pin theo commit hash (immutable — bất biến hoàn toàn)
module "vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//networking/vpc?ref=a1b2c3d4e5f6"
}
```

---

## 🔄 Upgrade Strategy — Chiến Lược Nâng Cấp

### Nâng Cấp Patch Version (v1.2.3 → v1.2.4)

```bash
# An toàn: chỉ sửa bug, không có breaking changes
# Quy trình:
# 1. Đọc CHANGELOG để xác nhận chỉ là bug fix
# 2. Cập nhật version trong code
# 3. Chạy terraform plan — không nên có resource replacement
# 4. Apply vào dev trước, sau đó staging, sau đó prod
```

```hcl
# Trước:
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.3"
}

# Sau khi đọc CHANGELOG và xác nhận an toàn:
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.4"
}
```

### Nâng Cấp Minor Version (v1.2.0 → v1.3.0)

```bash
# Tương đối an toàn: thêm tính năng mới backward-compatible
# Quy trình:
# 1. Đọc kỹ CHANGELOG — check cho "deprecated" warnings
# 2. Kiểm tra inputs/outputs có thay đổi không
# 3. Chạy terraform plan — có thể có thay đổi nhỏ
# 4. Test đầy đủ trên dev/staging trước khi prod
```

### Nâng Cấp Major Version (v1.x → v2.x) — Breaking Changes

```bash
# Nguy hiểm: có thể destroy và recreate resources
# Quy trình cẩn thận:
# 1. Đọc migration guide — Hướng dẫn di chuyển (nếu có)
# 2. Backup state file
# 3. Xem rõ các variables/outputs bị đổi tên hoặc xoá
# 4. Cập nhật code caller trước khi upgrade module version
# 5. Chạy terraform plan — kiểm tra kỹ "destroy" operations
# 6. Cân nhắc dùng "moved" blocks nếu resource bị rename
```

```hcl
# Ví dụ: Module VPC v1 → v2 đổi tên variable
# v1: var.vpc_cidr
# v2: var.cidr_block

# Bước 1: Update variable name trong caller
module "vpc" {
  source  = "my-org/vpc/aws"
  version = "2.0.0"    # Bump version

  # cidr_block thay cho vpc_cidr (breaking change)
  cidr_block = "10.0.0.0/16"
}

# Bước 2: Nếu resource bị rename, dùng moved block
moved {
  from = module.vpc.aws_vpc.vpc
  to   = module.vpc.aws_vpc.this
}
```

---

## 📋 CHANGELOG.md — Nhật Ký Thay Đổi

### Định Dạng Keep a Changelog

```markdown
# Changelog

Tất cả thay đổi quan trọng sẽ được ghi lại ở đây.

Định dạng theo [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
và project này tuân theo [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased] — Chưa Release

## [2.0.0] — 2026-04-15

### ⚠️ BREAKING CHANGES — Thay Đổi Phá Vỡ Tương Thích

- **Đổi tên variable:** `vpc_cidr` → `cidr_block` (#45)
- **Xoá output:** `vpc_arn` bị xoá (dùng `vpc_id` thay thế) (#47)
- **Yêu cầu Terraform >= 1.3** (trước đây >= 0.13) (#48)

### Migration Guide — Hướng Dẫn Di Chuyển

```hcl
# Trước (v1.x):
module "vpc" {
  vpc_cidr = "10.0.0.0/16"
}

# Sau (v2.x):
module "vpc" {
  cidr_block = "10.0.0.0/16"
}
```

## [1.2.0] — 2026-03-10

### Added — Thêm Mới

- Thêm support cho IPv6 CIDR block (`enable_ipv6 = true`)
- Thêm variable `flow_logs_config` để bật VPC Flow Logs (#40)
- Thêm output `vpc_ipv6_cidr_block` (#41)

### Fixed — Sửa Lỗi

- Sửa lỗi NAT Gateway không được tạo khi `single_nat_gateway = false` (#38)

## [1.1.0] — 2026-02-05

### Added — Thêm Mới

- Thêm variable `enable_dns_hostnames` (default: true)
- Thêm optional VPC Endpoint cho S3 (#30)

### Changed — Thay Đổi

- Cải thiện resource tagging để bao gồm AZ name (#32)

## [1.0.0] — 2026-01-01

### Initial Release — Phát Hành Lần Đầu

- VPC với public và private subnets
- Internet Gateway
- NAT Gateway (single hoặc per-AZ)
- Route tables
```

---

## 🔒 Module Lock File — File Khóa Module

### `.terraform.lock.hcl` Cho Providers

```hcl
# File này được tạo tự động bởi terraform init
# Nó lock hash của providers — KHÔNG phải modules từ git

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = ">= 4.0.0, ~> 5.0"
  hashes = [
    "h1:rgILSZi3MZUV7RsF0dvE7p21fB9/cHnHlkT4bGmqLKc=",
    # ... nhiều hashes cho các platforms khác nhau
  ]
}
```

```bash
# Lock providers cho tất cả platforms (CI/CD + local dev)
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# Commit file vào git — đảm bảo mọi người dùng cùng provider version
git add .terraform.lock.hcl
git commit -m "chore: update provider lock file"
```

---

## 🛠️ Automated Version Management — Quản Lý Phiên Bản Tự Động

### Renovate Bot / Dependabot Cho Terraform

```yaml
# .github/dependabot.yml — Tự động tạo PR khi có version mới
version: 2
updates:
  - package-ecosystem: "terraform"
    directory: "/environments/dev"
    schedule:
      interval: "weekly"
    # Tự động approve patch updates
    # Yêu cầu review thủ công cho major updates
```

```json
// renovate.json — Cấu hình Renovate Bot nâng cao
{
  "terraform": {
    "enabled": true
  },
  "packageRules": [
    {
      "matchPackagePatterns": ["terraform-aws-modules/*"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    },
    {
      "matchPackagePatterns": ["terraform-aws-modules/*"],
      "matchUpdateTypes": ["major"],
      "labels": ["breaking-change", "needs-review"]
    }
  ]
}
```

---

## 📊 So Sánh Version Constraint Patterns

| Pattern        | Ví Dụ            | Cho Phép                      | Khi Nào Dùng                      |
| -------------- | ---------------- | ----------------------------- | --------------------------------- |
| Exact          | `"1.2.3"`        | Chỉ 1.2.3                    | Production critical                |
| Pessimistic    | `"~> 1.2"`       | >= 1.2, < 2.0                | Module tin cậy, theo minor        |
| Pessimistic    | `"~> 1.2.3"`     | >= 1.2.3, < 1.3.0            | Module tin cậy, theo patch        |
| Range          | `">= 1.2, < 2"`  | 1.2.0 đến 1.x.x              | Kiểm soát explicit                |
| Min only       | `">= 1.0"`       | 1.0.0 trở lên               | Development/testing               |
| Git tag        | `?ref=v1.2.3`    | Exact tag                     | Git-based private modules         |
| Git hash       | `?ref=abc1234`   | Exact commit (immutable)      | Security-critical modules         |

---

## ✅ Checklist Version Management

```
Khi phát triển module:
  [ ] Dùng semantic versioning cho git tags
  [ ] Cập nhật CHANGELOG.md trước khi tag
  [ ] Ghi rõ BREAKING CHANGES trong CHANGELOG và README
  [ ] Cung cấp migration guide cho major version bumps
  [ ] Test module với versions mới trước khi release

Khi dùng module trong project:
  [ ] Luôn pin version (không bao giờ để mà không có version)
  [ ] Dùng exact version cho production
  [ ] Commit .terraform.lock.hcl vào git
  [ ] Có quy trình review khi upgrade module version
  [ ] Kiểm tra terraform plan sau mỗi version upgrade

CI/CD:
  [ ] Dùng Renovate Bot hoặc Dependabot để theo dõi updates
  [ ] Có stage test upgrade trước khi apply production
  [ ] Auto-approve patch updates sau khi tests pass
  [ ] Require manual review cho major version upgrades
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Tiếp Theo:** [5-composition-patterns.md](5-composition-patterns.md) — Flat, Nested, Wrapper module patterns
