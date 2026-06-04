# 5 — Chiến Lược Kiểm Thử Toàn Diện — Comprehensive Testing Strategy

> Không có công cụ đơn lẻ nào đủ. Chiến lược kiểm thử hạ tầng tốt là kết hợp đúng công cụ ở đúng giai đoạn — nhanh và rẻ ở đầu pipeline, sâu và đắt ở cuối.

---

## Tổng Quan Chiến Lược

### Defense in Depth — Bảo Vệ Theo Chiều Sâu

Mượn khái niệm từ bảo mật, "defense in depth" trong testing nghĩa là mỗi tầng kiểm thử bắt một loại lỗi khác nhau, không phụ thuộc vào tầng khác:

```
Developer Machine          CI Pipeline (< 5 phút)     CD Pipeline (< 30 phút)
─────────────────          ───────────────────────     ───────────────────────
terraform fmt              terraform fmt -check        terraform plan
terraform validate         terraform validate          Checkov (full)
tflint (nhanh)             tflint --recursive          Conftest (OPA policies)
checkov -d . (quick)       Checkov + tfsec             Terratest (module tests)
                           Infracost comment           Manual approval gate
                                                       terraform apply
```

---

## Giai Đoạn 1: Pre-commit — Trước Khi Commit

**Mục tiêu:** Bắt lỗi ngay trên máy developer, không để lên remote.

### Thiết Lập Pre-commit Framework

```yaml
# .pre-commit-config.yaml
repos:
  # Kiểm tra Terraform cơ bản
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_fmt          # Tự động định dạng
      - id: terraform_validate     # Kiểm tra cú pháp
      - id: terraform_tflint       # Linting nâng cao
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl

      - id: checkov                # Security quick scan
        args:
          - --args=--quiet
          - --args=--framework=terraform

  # Kiểm tra chung
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace    # Xóa khoảng trắng thừa
      - id: end-of-file-fixer      # Đảm bảo newline ở cuối file
      - id: check-merge-conflict   # Phát hiện merge conflict markers
      - id: detect-private-key     # Phát hiện private key trong code
```

```bash
# Cài đặt một lần
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg  # Kiểm tra commit message

# Test tất cả hook mà không cần commit
pre-commit run --all-files
```

### Script Quick Check Thủ Công

```bash
#!/bin/bash
# scripts/check.sh — Chạy trước khi push

set -e   # Dừng ngay khi có lỗi — Exit on error

echo "🔍 Terraform Format Check..."
terraform fmt -recursive -check

echo "✅ Terraform Validate..."
terraform init -backend=false -upgrade
terraform validate

echo "🔎 TFLint..."
tflint --recursive

echo "🔒 Checkov Security Scan..."
checkov -d . --quiet --framework terraform

echo "✅ Tất cả kiểm tra đã qua!"
```

---

## Giai Đoạn 2: Pull Request CI — Tích Hợp Liên Tục Trên PR

**Mục tiêu:** Block PR nếu vi phạm policy, cung cấp thông tin cho reviewer.

### GitHub Actions Pipeline Đầy Đủ

```yaml
# .github/workflows/terraform-ci.yml
name: Terraform CI

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'
      - '.github/workflows/terraform-ci.yml'

env:
  TF_VERSION: "1.9.0"
  AWS_REGION: "us-east-1"

jobs:
  # Job 1: Static Analysis — Phân tích tĩnh (chạy song song)
  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive -diff

      - name: Terraform Validate
        run: |
          terraform init -backend=false
          terraform validate

  # Job 2: Linting (chạy song song với static-analysis)
  lint:
    name: TFLint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@v4

      - name: TFLint Init
        run: tflint --init
        env:
          GITHUB_TOKEN: ${{ github.token }}

      - name: Run TFLint
        run: tflint --recursive --format=compact

  # Job 3: Security Scan (chạy song song)
  security:
    name: Security & Compliance
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
          skip_check: CKV_AWS_117,CKV_AWS_118   # Exception có phê duyệt

      - name: Upload Security Scan Results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif

  # Job 4: Terraform Plan với cost estimate (cần credentials)
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: [static-analysis, lint, security]   # Chỉ chạy sau khi 3 job trên pass
    permissions:
      pull-requests: write    # Để comment plan output lên PR
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        id: plan
        run: terraform plan -out=tfplan.binary -no-color
        continue-on-error: true

      - name: Comment Plan Output on PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📋
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pushed by: @${{ github.actor }}*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Infracost Cost Estimate
        run: |
          terraform show -json tfplan.binary > tfplan.json
          infracost diff --path=tfplan.json --format=comment \
            --github-token=${{ github.token }} \
            --pull-request=${{ github.event.pull_request.number }} \
            --repo=${{ github.repository }}
```

---

## Giai Đoạn 3: Merge to Main — Kiểm Thử Sau Khi Merge

**Mục tiêu:** Chạy integration tests thực tế, chuẩn bị cho deploy.

```yaml
# .github/workflows/terraform-integration-test.yml
name: Terraform Integration Tests

on:
  push:
    branches: [main]
    paths:
      - 'modules/**'

jobs:
  integration-test:
    name: Terratest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
          cache: true

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"
          terraform_wrapper: false   # Quan trọng cho Terratest

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_TEST }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_TEST }}
          aws-region: us-east-1
          role-to-assume: ${{ secrets.TERRAFORM_TEST_ROLE_ARN }}

      - name: Run Integration Tests
        run: |
          cd test/
          go test -v -timeout 45m -run TestVPCModule ./...
          go test -v -timeout 45m -run TestWebServerModule ./...
        env:
          TF_VAR_environment: test
          TF_VAR_suffix: ${{ github.sha }}

      - name: Cleanup on Failure
        if: failure()
        run: |
          cd test/
          # Script tìm và destroy resource test còn sót
          aws resourcegroupstaggingapi get-resources \
            --tag-filters Key=ManagedBy,Values=Terratest \
            --query 'ResourceTagMappingList[].ResourceARN' \
            --output text | xargs -I {} aws ec2 terminate-instances --instance-ids {}
```

---

## Ma Trận Công Cụ — Công Cụ Nào Cho Mục Đích Gì

| Công Cụ | Phát Hiện | Khi Chạy | Tốc Độ | Chi Phí Cloud |
|---------|-----------|----------|--------|--------------|
| `terraform fmt` | Định dạng code | Pre-commit, CI | < 1s | $0 |
| `terraform validate` | Cú pháp HCL, tham chiếu | Pre-commit, CI | < 5s | $0 |
| TFLint | Provider validation, naming | Pre-commit, CI | < 30s | $0 |
| Checkov | Security misconfig, compliance | Pre-commit, CI | < 2m | $0 |
| tfsec | Security vulnerabilities | CI | < 1m | $0 |
| Infracost | Ước tính chi phí | CI (trên plan) | < 1m | $0 |
| OPA/Conftest | Custom policies | CI (trên plan JSON) | < 1m | $0 |
| terraform test | Module behavior (plan-only) | CI | < 5m | $0* |
| Terratest | Integration testing thực tế | CD / main branch | 10-60m | $0.01-5.00 |

*terraform test có thể dùng `command = apply` thì tốn chi phí

---

## Chiến Lược Theo Quy Mô Team

### Team Nhỏ (1-5 người)

```
Ưu tiên: Pre-commit hooks + GitHub Actions cơ bản
Công cụ: fmt + validate + TFLint + Checkov
Không cần: Terratest (tốn thời gian setup)
Thời gian CI: < 5 phút
```

### Team Trung Bình (5-20 người)

```
Ưu tiên: Đầy đủ CI + Integration test cho module quan trọng
Công cụ: fmt + validate + TFLint + Checkov + Infracost + Terratest cho VPC/Security modules
Thêm: OPA policy cho custom business rules
Thời gian CI: < 10 phút
```

### Team Lớn / Enterprise (20+ người)

```
Ưu tiên: Fully automated với manual approval gate
Công cụ: Toàn bộ stack + Sentinel (Terraform Cloud/Enterprise)
Thêm: Drift detection schedule, cost budget alerts
Thời gian CI: < 15 phút (song song)
Thêm: Dedicated test AWS account
```

---

## Testing Cho Terraform Modules Cụ Thể

### Module Network — Mạng (VPC, Subnet)

```
Priority: Cao
Tests:
  ✅ fmt + validate + TFLint
  ✅ Checkov (security groups, NACLs, flow logs)
  ✅ Terratest: VPC tồn tại, CIDR đúng, route table đúng
  ✅ Terratest: Connectivity test giữa subnets
```

### Module Database — Cơ Sở Dữ Liệu (RDS, ElastiCache)

```
Priority: Rất cao
Tests:
  ✅ fmt + validate + TFLint
  ✅ Checkov: encryption at rest, backup enabled, no public access
  ✅ Terratest: Instance status available, multi-AZ, encryption
  ⚠️ KHÔNG test connectivity trực tiếp từ CI (network boundary)
```

### Module Compute — Tính Toán (EC2, ECS, Lambda)

```
Priority: Cao
Tests:
  ✅ fmt + validate + TFLint
  ✅ Checkov: IMDSv2, security group rules, IAM role scope
  ✅ Terratest: Instance running, health check passing
  ✅ HTTP test nếu có load balancer
```

### Module IAM — Quản Lý Quyền Truy Cập

```
Priority: Rất cao
Tests:
  ✅ fmt + validate
  ✅ Checkov: no wildcard, no privilege escalation
  ✅ OPA: custom policy cho naming convention và scope
  ⚠️ Manual review bắt buộc cho thay đổi IAM
```

---

## Xử Lý False Positives — Kết Quả Dương Tính Giả

False positive — Kết quả dương tính giả — là khi tool báo lỗi nhưng thực tế không phải lỗi. Quản lý chặt chẽ để không "normalize" việc bỏ qua warnings:

```bash
# 1. Document lý do skip
# checkov:skip=CKV_AWS_117:Lambda trong VPC private, không cần VPC attachment

# 2. Track exceptions trong file tập trung
# .checkov.yaml
skip-check:
  - id: CKV_AWS_117
    reason: "Lambda functions được deploy trong private VPC, không cần VPC endpoint riêng"
    approved-by: "security-team"
    approved-date: "2026-03-15"
    review-date: "2026-09-15"   # Nhắc review lại sau 6 tháng
```

---

## KPIs — Key Performance Indicators — Chỉ Số Đo Lường Chất Lượng

Đo lường hiệu quả của testing strategy:

| KPI | Mục Tiêu | Cách Đo |
|-----|---------|---------|
| Test pipeline time | < 10 phút | GitHub Actions duration |
| Security issue escape rate | 0 critical/high | Checkov results in prod |
| Test coverage | 100% modules có fmt+validate+Checkov | Script đếm module thiếu test |
| False positive rate | < 5% | Số skip annotation / total checks |
| Mean time to detect | < 1 giờ | Thời gian từ commit đến phát hiện issue |

---

## Câu Hỏi Phỏng Vấn

**Q: Mô tả testing strategy cho Terraform trong team của bạn?**

A: Chúng tôi dùng 3 lớp. Lớp 1 là pre-commit hooks chạy `fmt`, `validate`, TFLint và Checkov quick scan — dưới 1 phút. Lớp 2 là CI pipeline trên mọi PR: static analysis song song với security scan, sau đó terraform plan với Infracost cost estimate và OPA policy check trên plan output. Lớp 3 là integration tests bằng Terratest chỉ chạy khi merge vào main, tập trung vào module VPC và security group vì chúng ảnh hưởng nhiều nhất.

**Q: Làm sao balance giữa test coverage và tốc độ pipeline?**

A: Test nhanh và rẻ chạy nhiều nhất — fmt, validate, TFLint trong < 30 giây. Test tốn kém như Terratest chạy có điều kiện — chỉ khi file trong `modules/` thay đổi, hoặc chỉ trên nhánh main. Chạy Terratest song song với `t.Parallel()` cũng giúp cắt giảm thời gian đáng kể. Mục tiêu: 80% lỗi được bắt trong 5 phút đầu của pipeline.

**Q: Khi nào nên dùng `terraform test` built-in thay vì Terratest?**

A: `terraform test` tốt cho module đơn giản và team không biết Go — bạn viết test bằng HCL, dễ maintain. Dùng Terratest khi cần test hành vi thực tế phức tạp: kiểm tra HTTP endpoint, SSH vào instance, test database connection, hoặc test end-to-end flow nhiều resource. `terraform test` cũng có thể chạy ở mode `plan` — không tốn chi phí — phù hợp cho most unit tests.

---

## Tóm Tắt — Bắt Đầu Từ Đâu?

```
Tuần 1:  Thêm terraform fmt + validate vào CI
Tuần 2:  Thêm TFLint với AWS plugin
Tuần 3:  Thêm Checkov với built-in rules
Tuần 4:  Review và thêm exception management
Tháng 2: Thêm Infracost cho cost visibility
Tháng 3: Viết Terratest cho module quan trọng nhất
Tháng 4: Thêm OPA policies cho business rules
```

**Nguyên tắc vàng:** Một pipeline không bao giờ fail vì noise — false positive — thì team sẽ trust và không bypass. Quản lý exception cẩn thận quan trọng hơn có nhiều tool.

---

**Kết Thúc Topic 07-testing** | Quay lại: [README.md](README.md) | Tiếp theo: [08-monitoring/](../08-monitoring/)
