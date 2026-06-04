# 4 — Static Analysis — Phân Tích Tĩnh Terraform Code

> SAST — Static Application Security Testing — Kiểm Thử Bảo Mật Ứng Dụng Tĩnh: phát hiện lỗ hổng trước khi deploy, không phải sau.

---

## 🎯 Tại Sao Cần Phân Tích Tĩnh?

Không có công cụ phân tích tĩnh, những lỗi sau dễ lọt qua:

```hcl
# Lỗi 1: Security group mở toàn bộ internet
resource "aws_security_group_rule" "allow_all" {
  cidr_blocks = ["0.0.0.0/0"]
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  type        = "ingress"
}

# Lỗi 2: S3 bucket công khai
resource "aws_s3_bucket_acl" "public" {
  acl = "public-read"  # Dữ liệu ai cũng đọc được
}

# Lỗi 3: RDS không mã hoá
resource "aws_db_instance" "main" {
  storage_encrypted = false  # Dữ liệu không mã hoá
}
```

---

## 🛠️ Ba Công Cụ Chính

### tfsec — Terraform Security Scanner

tfsec là công cụ SAST chuyên cho Terraform, kiểm tra hàng trăm rules bảo mật.

#### Cài Đặt

```bash
# macOS
brew install tfsec

# Linux
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Docker
docker pull aquasec/tfsec:latest
```

#### Chạy Cơ Bản

```bash
# Scan thư mục hiện tại
tfsec .

# Scan thư mục cụ thể
tfsec ./infrastructure/terraform

# Output JSON (cho CI/CD parsing)
tfsec --format json . > tfsec-results.json

# Chỉ show HIGH và CRITICAL
tfsec --minimum-severity HIGH .

# Bỏ qua specific rules
tfsec --exclude aws-s3-enable-bucket-encryption .
```

#### Output Mẫu

```
Result #1 HIGH Security group rule allows ingress from public internet.
───────────────────────────────────────────
  /terraform/main.tf:15-22
───────────────────────────────────────────
   15 │ resource "aws_security_group_rule" "allow_ssh" {
   16 │   cidr_blocks = ["0.0.0.0/0"]   ← Vấn đề tại đây
   17 │   from_port   = 22
   18 │   to_port     = 22
───────────────────────────────────────────
  Rule ID:  aws-ec2-no-public-ingress-sgr
  Impact:   SSH port open to the public internet
  Fix:      Restrict cidr_blocks to specific IP ranges
  Link:     https://aquasecurity.github.io/tfsec/...
```

#### Custom Checks — Kiểm Tra Tùy Chỉnh

```yaml
# .tfsec/custom_checks.yaml
checks:
  - code: CUS001
    description: "Ensure all EC2 instances have required tags"
    impact: "Missing tags make cost allocation difficult"
    resolution: "Add required tags to all EC2 instances"
    requiredTypes:
      - resource
    requiredLabels:
      - aws_instance
    severity: MEDIUM
    matchSpec:
      name: tags
      action: contains
      value: "Environment"
```

---

### Checkov — Compliance Scanner Đa Nền Tảng

Checkov kiểm tra compliance — tuân thủ — với các frameworks như CIS, PCI-DSS, SOC2, HIPAA.

#### Cài Đặt

```bash
# pip
pip install checkov

# Docker
docker pull bridgecrew/checkov
```

#### Chạy Cơ Bản

```bash
# Scan Terraform directory
checkov -d .

# Scan file cụ thể
checkov -f main.tf

# Chỉ check một số frameworks
checkov -d . --framework terraform

# Output với chi tiết
checkov -d . --output cli --output json --output-file-path ./results

# Chỉ show FAILED checks
checkov -d . --compact

# Skip một số checks
checkov -d . --skip-check CKV_AWS_20,CKV_AWS_57
```

#### Output Mẫu

```
Check: CKV_AWS_20: "Ensure the S3 bucket has access control list (ACL) applied"
FAILED for resource: aws_s3_bucket.data_bucket
File: /terraform/s3.tf:1-10

	1 | resource "aws_s3_bucket" "data_bucket" {
	2 |   bucket = "my-data-bucket"
	3 |   acl    = "public-read"    ← Vấn đề
	4 | }

Guide: https://docs.bridgecrew.io/docs/s3_1-acl-check

Passed checks: 45, Failed checks: 3, Skipped checks: 0
```

#### Checkov Với Compliance Frameworks

```bash
# Kiểm tra theo CIS AWS Benchmark
checkov -d . --framework terraform --check CIS_AWS

# Kiểm tra theo PCI-DSS — Payment Card Industry Data Security Standard
checkov -d . --framework terraform --check PCI

# Kiểm tra theo SOC2 — Service Organization Control 2
checkov -d . --framework terraform --check SOC2

# Tạo báo cáo SARIF — Static Analysis Results Interchange Format
checkov -d . --output sarif --output-file results.sarif
```

#### Custom Policy Với Python

```python
# custom_check.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class EnsureTagEnvironment(BaseResourceCheck):
    def __init__(self):
        name = "Ensure EC2 has Environment tag"
        id = "CKV_CUSTOM_1"
        supported_resources = ["aws_instance"]
        categories = [CheckCategories.GENERAL_SECURITY]
        super().__init__(name=name, id=id,
                         categories=categories,
                         supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        tags = conf.get("tags", [{}])[0]
        if isinstance(tags, dict) and "Environment" in tags:
            return CheckResult.PASSED
        return CheckResult.FAILED

scanner = EnsureTagEnvironment()
```

---

### Terrascan — Policy-as-Code Scanner

Terrascan dùng OPA — Open Policy Agent — Tác Nhân Chính Sách Mở để define policies.

#### Cài Đặt

```bash
# macOS
brew install terrascan

# Binary download
curl -L "https://github.com/tenable/terrascan/releases/latest/download/terrascan_Linux_x86_64.tar.gz" | tar -xz
```

#### Chạy

```bash
# Scan Terraform
terrascan scan -t terraform -d .

# Output JSON
terrascan scan -t terraform -d . -o json

# Scan với custom policies
terrascan scan -t terraform -d . --policy-path ./custom-policies
```

#### Custom Policy Bằng Rego — Ngôn Ngữ Rego

```rego
# policies/require_encryption.rego
package accurics.terraform.encryption.require_kms

# Deny nếu RDS instance không bật encryption
deny[msg] {
  resource := input.resource.aws_db_instance[name]
  not resource.config.storage_encrypted

  msg := sprintf(
    "RDS instance '%s' must have storage_encrypted = true",
    [name]
  )
}
```

---

## 🔄 Tích Hợp Vào CI/CD Pipeline

### GitHub Actions — Chạy Tất Cả Công Cụ

```yaml
# .github/workflows/terraform-security.yml
name: Terraform Security Scan

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  tfsec:
    name: tfsec — Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false              # Fail pipeline nếu có findings
          minimum_severity: HIGH        # Chỉ fail với HIGH+
          additional_args: --format sarif --out tfsec.sarif

      - name: Upload SARIF to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: tfsec.sarif

  checkov:
    name: Checkov — Compliance Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false
          skip_check: CKV_AWS_144  # Skip nếu không applicable

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif
```

### GitLab CI

```yaml
# .gitlab-ci.yml
terraform-security:
  stage: validate
  image: aquasec/tfsec:latest
  script:
    - tfsec --minimum-severity HIGH --format json . > tfsec-report.json
    - |
      # Fail nếu có CRITICAL findings
      CRITICAL=$(cat tfsec-report.json | jq '.results[] | select(.severity == "CRITICAL")' | jq -s length)
      if [ "$CRITICAL" -gt "0" ]; then
        echo "Found $CRITICAL CRITICAL security issues!"
        cat tfsec-report.json | jq '.results[] | select(.severity == "CRITICAL")'
        exit 1
      fi
  artifacts:
    reports:
      sast: tfsec-report.json
    paths:
      - tfsec-report.json
    when: always

checkov-scan:
  stage: validate
  image: bridgecrew/checkov:latest
  script:
    - checkov -d . --framework terraform --output cli --output json
      --output-file-path ./checkov-results --compact
  artifacts:
    paths:
      - checkov-results/
    when: always
```

---

## ⚙️ Cấu Hình Ngoại Lệ — Suppressing Findings

### tfsec Inline Suppression

```hcl
resource "aws_security_group_rule" "bastion_ssh" {
  # tfsec:ignore:aws-ec2-no-public-ingress-sgr
  cidr_blocks = ["0.0.0.0/0"]
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  type        = "ingress"
  description = "Bastion host SSH — intentionally public, protected by key pair"
}
```

### .tfsec/config.yaml

```yaml
# .tfsec/config.yaml
severity_overrides:
  aws-ec2-no-public-ingress-sgr: LOW   # Hạ mức độ

exclude:
  - aws-s3-enable-bucket-encryption    # Skip hẳn
```

### Checkov Inline Suppression

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-access-logs"
  # checkov:skip=CKV_AWS_20:This bucket intentionally has no ACL restrictions for log delivery
  # checkov:skip=CKV_AWS_144:Cross-region replication not needed for logs
}
```

### .checkov.yaml

```yaml
# .checkov.yaml
skip-check:
  - CKV_AWS_144    # S3 cross-region replication
  - CKV_AWS_145    # S3 default encryption (we use bucket encryption instead)

# Chỉ check specific frameworks
check:
  - CIS_AWS_1.4
  - PCI

# Fail nếu có HIGH hoặc CRITICAL
hard-fail-on: CRITICAL,HIGH
```

---

## 📊 So Sánh Công Cụ

| Tính Năng                      | tfsec    | Checkov  | Terrascan |
| ------------------------------ | -------- | -------- | --------- |
| Terraform support              | ✅       | ✅       | ✅        |
| Multi-platform (K8s, Docker)   | ❌       | ✅       | ✅        |
| Custom policies                | YAML     | Python   | Rego/OPA  |
| Compliance frameworks          | Partial  | ✅       | ✅        |
| SARIF output                   | ✅       | ✅       | ✅        |
| GitHub integration             | ✅       | ✅       | ✅        |
| Speed                          | Nhanh    | Trung    | Trung     |
| Community rules                | Nhiều    | Rất nhiều| Trung     |

**Khuyến nghị:** Dùng **tfsec** cho security-focused scan và **Checkov** cho compliance. Kết hợp cả hai cho coverage tốt nhất.

---

## 🎯 Mức Độ Nghiêm Trọng Và Ngưỡng Fail

```bash
# Chiến lược thực tế:
# - CRITICAL: Block merge ngay lập tức
# - HIGH: Block merge, cần fix trước khi merge
# - MEDIUM: Warning, cần fix trước khi merge vào main
# - LOW: Thông tin, không block

# tfsec — block nếu có HIGH+
tfsec --minimum-severity HIGH --tfvars-file terraform.tfvars .

# Checkov — soft fail cho MEDIUM trở xuống
checkov -d . --soft-fail-on MEDIUM,LOW
```

---

## ✅ Checklist Static Analysis

- [ ] tfsec được tích hợp vào CI/CD pipeline
- [ ] Checkov được tích hợp vào CI/CD pipeline
- [ ] Pipeline block merge khi có CRITICAL/HIGH findings
- [ ] SARIF results được upload lên GitHub Security tab
- [ ] Custom policies cho organizational rules đã được viết
- [ ] Suppression comments có lý do rõ ràng
- [ ] Scan chạy trên mọi PR có thay đổi `.tf` files
- [ ] Results được review trong pull request comments

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: tfsec và Checkov khác nhau như thế nào?**
> tfsec tập trung vào security misconfigurations đặc trưng của Terraform (security groups, encryption, public access). Checkov rộng hơn: hỗ trợ nhiều platform (Terraform, CloudFormation, K8s, Dockerfile) và map findings sang compliance frameworks (CIS, PCI-DSS, SOC2). Trong thực tế, dùng cả hai cho coverage toàn diện.

**Q: Làm sao tích hợp vào CI/CD mà không block mọi thứ?**
> Dùng tiered severity: CRITICAL/HIGH block pipeline ngay, MEDIUM là warning không block nhưng phải có ticket để fix, LOW chỉ là thông tin. Dùng `--soft-fail` cho MEDIUM/LOW để không block nhưng vẫn xuất hiện trong report. Cấu hình exception rõ ràng với comment giải thích lý do.

**Q: Khi nào nên bỏ qua một security finding?**
> Khi có lý do kỹ thuật chính đáng (bastion host cần SSH public, legacy system), đã có biện pháp bù đắp khác (MFA, IP allowlist, network ACL), và được review bởi security team. Phải có comment giải thích ngay trong code (`tfsec:ignore` với lý do) và được tracking trong issue tracker.
