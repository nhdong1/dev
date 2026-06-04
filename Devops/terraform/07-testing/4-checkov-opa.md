# 4 — Checkov & OPA — Open Policy Agent — Compliance Testing — Kiểm Thử Tuân Thủ

> Checkov và OPA — Open Policy Agent — Tác Nhân Chính Sách Mở — là hai công cụ bổ sung cho nhau để kiểm tra hạ tầng Terraform có tuân thủ các chuẩn bảo mật và chính sách tổ chức không, trước khi bất kỳ tài nguyên nào được tạo.

---

## Tại Sao Cần Compliance Testing?

Static analysis — Phân tích tĩnh — như `terraform validate` kiểm tra code hợp lệ về mặt cú pháp. Compliance testing kiểm tra code có **đúng về mặt bảo mật và chính sách** không:

| Loại Kiểm Tra | validate | TFLint | Checkov/OPA |
|--------------|---------|--------|-------------|
| Cú pháp HCL | ✅ | ✅ | ❌ |
| Instance type hợp lệ | ❌ | ✅ | ❌ |
| S3 bucket có bật encryption? | ❌ | ❌ | ✅ |
| Security group mở port 22 ra internet? | ❌ | ❌ | ✅ |
| Tất cả resource có tag `CostCenter`? | ❌ | ❌ | ✅ |
| Chính sách nội bộ tùy chỉnh? | ❌ | ❌ | ✅ (OPA) |

---

## Checkov — Công Cụ Compliance Toàn Diện

### Checkov Là Gì?

Checkov là công cụ SAST — Static Application Security Testing — Kiểm Thử Bảo Mật Ứng Dụng Tĩnh — cho IaC — Infrastructure as Code. Nó quét file Terraform, CloudFormation, Kubernetes, Dockerfile, và nhiều hơn nữa, so sánh với hàng nghìn rule từ các chuẩn:

- **CIS Benchmarks** — Center for Internet Security — Tiêu Chuẩn Bảo Mật Internet
- **HIPAA** — Health Insurance Portability and Accountability Act — Luật Bảo Vệ Thông Tin Y Tế
- **PCI-DSS** — Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Thẻ Thanh Toán
- **SOC2** — Service Organization Control 2 — Kiểm Soát Tổ Chức Dịch Vụ
- **NIST** — National Institute of Standards and Technology — Viện Tiêu Chuẩn Công Nghệ Quốc Gia

### Cài Đặt Checkov

```bash
# Cài qua pip
pip install checkov

# Cài qua brew (macOS)
brew install checkov

# Dùng Docker
docker run --rm -v $(pwd):/tf bridgecrew/checkov -d /tf

# Kiểm tra version
checkov --version
```

### Cách Chạy Checkov

```bash
# Quét thư mục hiện tại
checkov -d .

# Quét file cụ thể
checkov -f main.tf

# Chỉ hiện failed checks — kiểm tra thất bại
checkov -d . --quiet

# Xuất kết quả JSON
checkov -d . -o json > checkov-report.json

# Xuất kết quả JUnit XML (cho CI/CD)
checkov -d . -o junitxml > checkov-report.xml

# Chỉ kiểm tra theo framework cụ thể
checkov -d . --framework terraform

# Bỏ qua check ID cụ thể
checkov -d . --skip-check CKV_AWS_18,CKV_AWS_21
```

### Ví Dụ Output

```
$ checkov -d .

terraform scan results:

Passed checks: 45, Failed checks: 3, Skipped checks: 0

Check: CKV_AWS_18: "Ensure the S3 bucket has access logging enabled"
	FAILED for resource: aws_s3_bucket.data
	File: /main.tf:10-20

Check: CKV_AWS_21: "Ensure all data stored in the S3 bucket have versioning enabled"
	FAILED for resource: aws_s3_bucket.data
	File: /main.tf:10-20

Check: CKV_AWS_116: "Ensure that AWS Lambda function is configured for a Dead Letter Queue"
	FAILED for resource: aws_lambda_function.processor
	File: /lambda.tf:5-25
```

### Các Rule Quan Trọng

```bash
# S3 Security
CKV_AWS_18   # Access logging bật không
CKV_AWS_19   # Server-side encryption bật không
CKV_AWS_20   # Bucket không public
CKV_AWS_21   # Versioning bật không

# EC2 / Instance Security
CKV_AWS_8    # IMDS v2 — Instance Metadata Service version 2 — bật không
CKV_AWS_135  # EBS volume mã hoá không

# Security Groups
CKV_AWS_25   # Security group không mở SSH (port 22) ra 0.0.0.0/0
CKV_AWS_26   # Security group không mở RDP (port 3389) ra 0.0.0.0/0

# IAM — Identity and Access Management
CKV_AWS_40   # IAM policy không có wildcard (*) trong Action
CKV_AWS_49   # IAM role không có trust policy quá rộng

# RDS — Relational Database Service
CKV_AWS_17   # RDS không public
CKV_AWS_157  # RDS mã hoá at rest — khi lưu trữ
CKV_AWS_133  # RDS bật deletion protection — bảo vệ xóa
```

### Custom Checkov Rules — Luật Tùy Chỉnh

```python
# checkov/custom_checks/check_required_tags.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class CheckRequiredTags(BaseResourceCheck):
    def __init__(self):
        name = "Ensure required tags are present"
        id = "CKV_CUSTOM_1"
        categories = [CheckCategories.GENERAL_SECURITY]
        supported_resources = ['aws_instance', 'aws_s3_bucket', 'aws_rds_instance']
        super().__init__(name=name, id=id, categories=categories,
                         supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        required_tags = ["Environment", "CostCenter", "Owner"]
        tags = conf.get("tags", [{}])

        if not tags or not isinstance(tags[0], dict):
            return CheckResult.FAILED

        for required_tag in required_tags:
            if required_tag not in tags[0]:
                return CheckResult.FAILED

        return CheckResult.PASSED

scanner = CheckRequiredTags()
```

```bash
# Chạy với custom check
checkov -d . --external-checks-dir ./checkov/custom_checks/
```

---

## OPA — Open Policy Agent — Tác Nhân Chính Sách Mở

### OPA Là Gì?

OPA là engine chính sách đa dụng — General-purpose policy engine — dùng ngôn ngữ Rego để viết chính sách có thể áp dụng cho nhiều hệ thống: Kubernetes, Terraform, API authorization, microservices.

Trong ngữ cảnh Terraform, OPA thường dùng với **Conftest** — công cụ đọc file `.tf` hoặc `terraform plan -out` và chạy policy bằng Rego.

### Cài Đặt

```bash
# OPA binary
brew install opa

# Conftest (công cụ chạy OPA policy cho config files)
brew install conftest

# Kiểm tra
opa version
conftest --version
```

### Ngôn Ngữ Rego — Viết Policy

```rego
# policy/terraform/required_tags.rego

package terraform.required_tags

# Tên package — Package name — phải match với conftest config

required_tags := {"Environment", "CostCenter", "Owner"}

# Deny — Từ chối — nếu resource thiếu tag bắt buộc
deny[msg] {
    resource := input.resource.aws_instance[name]
    missing := required_tags - {tag | resource.values.tags[tag]}
    count(missing) > 0
    msg := sprintf(
        "aws_instance '%s' thiếu các tag bắt buộc: %v",
        [name, missing]
    )
}

# Deny nếu S3 bucket bật public access
deny[msg] {
    resource := input.resource.aws_s3_bucket[name]
    resource.values.acl == "public-read"
    msg := sprintf(
        "S3 bucket '%s' không được phép có ACL public-read",
        [name]
    )
}
```

### Cách Dùng Conftest Với Terraform

```bash
# Bước 1: Tạo terraform plan output dạng JSON
terraform init
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Bước 2: Chạy conftest
conftest test tfplan.json --policy ./policy/terraform/

# Output:
# FAIL - tfplan.json - terraform.required_tags - aws_instance 'web' thiếu các tag bắt buộc: {"CostCenter", "Owner"}
# FAIL - tfplan.json - terraform.required_tags - S3 bucket 'data' không được phép có ACL public-read

# 2 tests, 0 passed, 0 warnings, 2 failures
```

### Ví Dụ Policies Phổ Biến

```rego
# policy/terraform/security.rego
package terraform.security

# Không cho phép security group mở SSH ra internet
deny[msg] {
    resource := input.resource.aws_security_group_rule[name]
    resource.values.type == "ingress"
    resource.values.from_port <= 22
    resource.values.to_port >= 22
    resource.values.cidr_blocks[_] == "0.0.0.0/0"
    msg := sprintf(
        "Security group rule '%s' mở SSH (port 22) ra 0.0.0.0/0 — không được phép",
        [name]
    )
}

# Cảnh báo nếu dùng instance type không nằm trong danh sách approved
warn[msg] {
    resource := input.resource.aws_instance[name]
    approved_types := {"t3.micro", "t3.small", "t3.medium", "t3.large"}
    not approved_types[resource.values.instance_type]
    msg := sprintf(
        "aws_instance '%s' dùng instance type '%s' không trong danh sách được duyệt",
        [name, resource.values.instance_type]
    )
}
```

```rego
# policy/terraform/cost_control.rego
package terraform.cost_control

# Không cho phép RDS multi-AZ — Multi-Availability Zone — trong môi trường dev
deny[msg] {
    resource := input.resource.aws_db_instance[name]
    resource.values.tags.Environment == "dev"
    resource.values.multi_az == true
    msg := sprintf(
        "RDS instance '%s' trong môi trường dev không cần multi_az=true (tốn kém không cần thiết)",
        [name]
    )
}
```

---

## Tích Hợp Trong CI/CD Pipeline

### GitHub Actions — Kết Hợp Checkov + Conftest

```yaml
name: Terraform Compliance Check

on:
  pull_request:
    paths:
      - '**.tf'

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend=false

      - name: Run Checkov
        id: checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif      # Format cho GitHub Security tab
          output_file_path: checkov.sarif
          soft_fail: false          # Fail pipeline nếu có issue

      - name: Upload Checkov Results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif

      - name: Generate Terraform Plan JSON
        run: |
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Install Conftest
        run: |
          wget https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
          tar xzf conftest_0.46.0_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin/

      - name: Run Conftest OPA Policies
        run: conftest test tfplan.json --policy ./policy/
```

### Checkov Trong Pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.0.0
    hooks:
      - id: checkov
        args:
          - --quiet
          - --skip-check=CKV_AWS_117   # Bỏ qua check này vì đã có exception
```

---

## Quản Lý Exception — Ngoại Lệ

### Checkov Exceptions

```hcl
# Bỏ qua check cụ thể với comment annotation
resource "aws_s3_bucket" "public_assets" {
  bucket = "my-public-website-assets"

  # checkov:skip=CKV_AWS_20:Bucket này cố ý public cho static website hosting
  # checkov:skip=CKV_AWS_18:Access logging không cần cho static assets
}
```

### OPA Exceptions

```rego
# Danh sách exception được phê duyệt
approved_public_buckets := {
    "my-public-website-assets",
    "company-downloads"
}

deny[msg] {
    resource := input.resource.aws_s3_bucket[name]
    resource.values.acl == "public-read"
    # Không deny nếu nằm trong danh sách exception
    not approved_public_buckets[resource.values.bucket]
    msg := sprintf("S3 bucket '%s' không được phép public", [name])
}
```

---

## Checkov vs OPA — Khi Nào Dùng Cái Nào?

| Tiêu Chí | Checkov | OPA / Conftest |
|---------|---------|----------------|
| **Cài đặt ban đầu** | Dễ (pip install) | Phức tạp hơn |
| **Built-in rules** | 1000+ rules sẵn có | Phải tự viết |
| **Custom rules** | Python | Rego |
| **Đa hệ thống** | Chủ yếu IaC | Kubernetes, API, IaC, bất kỳ đâu |
| **Policy sharing** | Khó chia sẻ cross-team | Dễ tái sử dụng |
| **Phù hợp cho** | Bắt đầu nhanh, CIS/HIPAA/PCI | Chính sách tùy chỉnh phức tạp |

**Khuyến nghị:** Bắt đầu với Checkov cho coverage nhanh, thêm OPA/Conftest khi cần chính sách tùy chỉnh theo business requirement — Yêu cầu kinh doanh.

---

## Câu Hỏi Phỏng Vấn

**Q: Checkov khác Terratest ở điểm nào?**

A: Checkov là SAST — phân tích tĩnh, không deploy gì, chạy trong vài giây. Nó tìm misconfiguration — Cấu hình sai — tiềm ẩn bằng cách đọc file `.tf`. Terratest deploy thật lên cloud và kiểm tra hành vi runtime. Checkov chạy mọi PR, Terratest chạy có chọn lọc vì tốn thời gian và chi phí.

**Q: Tại sao dùng OPA thay vì chỉ dùng Checkov?**

A: Checkov có sẵn hàng nghìn rule tiêu chuẩn, nhưng mỗi tổ chức có chính sách riêng — ví dụ "tất cả RDS trong môi trường prod phải dùng instance class r6g trở lên". OPA với Rego cho phép viết chính sách tùy chỉnh phức tạp, tái sử dụng policy cho nhiều hệ thống (Terraform + Kubernetes + API gateway), và kiểm tra thông qua gitops workflow.

**Q: Policy as Code — Chính Sách Dưới Dạng Mã — là gì?**

A: Thay vì viết chính sách bảo mật và compliance trong tài liệu Word, Policy as Code là viết chúng dưới dạng code (Rego, Python, YAML) có thể version control — Kiểm soát phiên bản — review, test, và tự động enforce — Bắt buộc áp dụng — trong CI/CD pipeline. Lợi ích: chính sách luôn được áp dụng nhất quán, có audit trail — Vết kiểm tra, dễ cập nhật và review.

---

**Tiếp theo:** [5-test-strategy.md](5-test-strategy.md) — Chiến lược kiểm thử toàn diện
