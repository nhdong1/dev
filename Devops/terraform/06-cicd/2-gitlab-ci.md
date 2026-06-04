# GitLab CI/CD Pipeline cho Terraform

> GitLab CI/CD — Hệ Thống Tích Hợp & Triển Khai Liên Tục của GitLab — cung cấp pipeline tích hợp sẵn với GitLab repository, rất phổ biến trong môi trường doanh nghiệp tự host.

---

## 🎯 Mục Tiêu

- Hiểu cấu trúc `.gitlab-ci.yml` cho Terraform
- Thiết lập pipeline đầy đủ với các stages
- Cấu hình GitLab CI Variables — Biến môi trường — an toàn
- Dùng GitLab Environments và Manual Gates — Cổng phê duyệt thủ công
- Tích hợp Merge Request — MR — comments cho plan output

---

## 📐 Kiến Trúc Pipeline GitLab

```
┌─────────────────────────────────────────────────────────────┐
│                    .gitlab-ci.yml                           │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│ validate │   lint   │ security │   plan   │    apply       │
│          │          │          │          │                │
│ fmt check│ tflint   │ tfsec    │ plan     │ apply          │
│ validate │          │ checkov  │ post MR  │ (manual gate)  │
└──────────┴──────────┴──────────┴──────────┴────────────────┘
     Tự động trên MR                         Manual trên main
```

---

## 📄 .gitlab-ci.yml Đầy Đủ

### Cấu Hình Cơ Bản

```yaml
# .gitlab-ci.yml

# Stages — Các giai đoạn thực thi theo thứ tự
stages:
  - validate
  - lint
  - security
  - plan
  - apply

# Biến toàn cục
variables:
  TF_VERSION: "1.7.0"
  TF_ROOT: "${CI_PROJECT_DIR}/infrastructure"
  TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/production"
  # TF_VAR_* — Terraform sẽ tự đọc các biến bắt đầu bằng TF_VAR_
  TF_VAR_environment: "production"

# Image mặc định — Dùng image chính thức của HashiCorp
default:
  image:
    name: hashicorp/terraform:${TF_VERSION}
    entrypoint: [""]

# Cache providers giữa các jobs để tăng tốc
cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - ${TF_ROOT}/.terraform

# Template chung — dùng YAML anchors để tránh lặp lại
.terraform_init: &terraform_init
  before_script:
    - cd ${TF_ROOT}
    - terraform init
        -backend-config="address=${TF_ADDRESS}"
        -backend-config="lock=true"
        -backend-config="username=gitlab-ci-token"
        -backend-config="password=${CI_JOB_TOKEN}"
        -input=false

# ─────────────────────────────────────────
# STAGE 1: VALIDATE
# ─────────────────────────────────────────

fmt:
  stage: validate
  <<: *terraform_init
  script:
    - terraform fmt -check -recursive
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  allow_failure: true  # Cảnh báo nhưng không dừng pipeline

validate:
  stage: validate
  <<: *terraform_init
  script:
    - terraform validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ─────────────────────────────────────────
# STAGE 2: LINT
# ─────────────────────────────────────────

tflint:
  stage: lint
  image: ghcr.io/terraform-linters/tflint:latest
  before_script:
    - cd ${TF_ROOT}
    - tflint --init
  script:
    - tflint --format=compact
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: true

# ─────────────────────────────────────────
# STAGE 3: SECURITY
# ─────────────────────────────────────────

tfsec:
  stage: security
  image:
    name: aquasec/tfsec:latest
    entrypoint: [""]
  script:
    - tfsec ${TF_ROOT} --format=json --out=/tmp/tfsec-report.json || true
    - tfsec ${TF_ROOT} --minimum-severity=HIGH
  artifacts:
    when: always
    reports:
      # GitLab Security Dashboard tích hợp
      sast: /tmp/tfsec-report.json
    paths:
      - /tmp/tfsec-report.json
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

checkov:
  stage: security
  image:
    name: bridgecrew/checkov:latest
    entrypoint: [""]
  script:
    - checkov
        --directory ${TF_ROOT}
        --framework terraform
        --output json
        --output-file /tmp/checkov-report.json
        --soft-fail
  artifacts:
    when: always
    paths:
      - /tmp/checkov-report.json
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# ─────────────────────────────────────────
# STAGE 4: PLAN
# ─────────────────────────────────────────

plan:
  stage: plan
  <<: *terraform_init
  script:
    - terraform plan
        -input=false
        -out=${TF_ROOT}/tfplan
        -var-file="environments/${CI_ENVIRONMENT_NAME}.tfvars"
        2>&1 | tee /tmp/plan_output.txt

    # Đăng plan vào Merge Request comment
    - |
      if [ -n "$CI_MERGE_REQUEST_IID" ]; then
        PLAN_CONTENT=$(cat /tmp/plan_output.txt)
        # Giới hạn độ dài
        if [ ${#PLAN_CONTENT} -gt 10000 ]; then
          PLAN_CONTENT="${PLAN_CONTENT:0:10000}...(truncated)"
        fi

        COMMENT_BODY="## 📋 Terraform Plan\n\n\`\`\`\n${PLAN_CONTENT}\n\`\`\`"

        curl --silent --request POST \
          --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
          --header "Content-Type: application/json" \
          --data "{\"body\": \"${COMMENT_BODY}\"}" \
          "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
      fi
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
      - /tmp/plan_output.txt
    expire_in: 1 hour  # Plan cũ sau 1 giờ không dùng được nữa
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ─────────────────────────────────────────
# STAGE 5: APPLY
# ─────────────────────────────────────────

apply:
  stage: apply
  <<: *terraform_init
  script:
    - terraform apply -input=false -auto-approve ${TF_ROOT}/tfplan
  environment:
    name: production
    url: https://console.aws.amazon.com
  dependencies:
    - plan  # Lấy plan artifact từ job plan
  rules:
    # Chỉ apply trên branch main
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Yêu cầu click thủ công
  allow_failure: false
```

---

## 🔐 Quản Lý Biến — Variables Management

### GitLab CI/CD Variables

```
GitLab Project → Settings → CI/CD → Variables

Loại biến:
┌─────────────────┬──────────────────────────────────────────┐
│ Loại            │ Mô Tả                                    │
├─────────────────┼──────────────────────────────────────────┤
│ Variable        │ Chuỗi văn bản thông thường               │
│ File            │ Nội dung được ghi ra file tạm            │
│ Masked          │ Không hiện trong logs                    │
│ Protected       │ Chỉ dùng trên protected branches/tags    │
└─────────────────┴──────────────────────────────────────────┘

Các biến cần tạo:
├── AWS_ACCESS_KEY_ID         — Masked, Protected
├── AWS_SECRET_ACCESS_KEY     — Masked, Protected
├── AWS_DEFAULT_REGION        — Variable
├── GITLAB_TOKEN              — Masked (để post MR comments)
└── TF_VAR_db_password        — Masked, Protected
```

### Dùng GitLab Managed Terraform State

```yaml
# GitLab có Terraform State backend tích hợp sẵn
# Không cần S3 hay GCS riêng

variables:
  # URL đến GitLab Terraform state backend
  TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${CI_ENVIRONMENT_NAME}"

before_script:
  - terraform init
      -backend-config="address=${TF_ADDRESS}"
      -backend-config="lock=true"
      -backend-config="username=gitlab-ci-token"
      -backend-config="password=${CI_JOB_TOKEN}"
```

```hcl
# terraform/backend.tf — Backend configuration
terraform {
  backend "http" {
    # Các giá trị được inject qua CI variables
    # Không hardcode ở đây
  }
}
```

---

## 🌍 Multi-Environment Pipeline

### Strategy: Branch = Environment

```yaml
# .gitlab-ci.yml — Phiên bản đa môi trường

# Template plan cho mọi môi trường
.plan_template: &plan_template
  stage: plan
  <<: *terraform_init
  script:
    - |
      terraform plan \
        -input=false \
        -out=${TF_ROOT}/tfplan \
        -var-file="environments/${ENV_NAME}.tfvars"
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 hour

# Template apply
.apply_template: &apply_template
  stage: apply
  <<: *terraform_init
  script:
    - terraform apply -input=false -auto-approve ${TF_ROOT}/tfplan
  dependencies:
    - plan_${ENV_NAME}  # Phụ thuộc vào plan tương ứng

# ─── Dev Environment ───
plan_dev:
  <<: *plan_template
  variables:
    ENV_NAME: dev
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/dev"
  environment:
    name: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

apply_dev:
  <<: *apply_template
  variables:
    ENV_NAME: dev
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/dev"
  environment:
    name: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
      when: on_success  # Tự động sau khi plan pass

# ─── Staging Environment ───
plan_staging:
  <<: *plan_template
  variables:
    ENV_NAME: staging
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/staging"
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

apply_staging:
  <<: *apply_template
  variables:
    ENV_NAME: staging
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/staging"
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Phải click thủ công

# ─── Production Environment ───
plan_production:
  <<: *plan_template
  variables:
    ENV_NAME: production
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/production"
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Plan cũng là manual để kiểm soát chặt hơn

apply_production:
  <<: *apply_template
  variables:
    ENV_NAME: production
    TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/production"
  environment:
    name: production
  needs:
    - plan_production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

---

## 🔒 GitLab Protected Environments

```
GitLab Project → Settings → CI/CD → Protected Environments

Cấu hình production:
├── Environment: production
├── Allowed to deploy: Developer role hoặc cụ thể user
├── Required approvals: 2 (2 người phải approve trước khi chạy)
└── Approval rules:
    ├── Group: platform-team (ít nhất 1 người)
    └── Group: security-team (ít nhất 1 người)
```

---

## 🛡️ Bảo Mật Pipeline

### Ngăn Chặn Secret Leakage — Rò Rỉ Bí Mật

```yaml
# Cấu hình job để không log sensitive values
apply:
  stage: apply
  variables:
    # Terraform không log giá trị sensitive
    TF_LOG: "WARN"  # Không dùng DEBUG trên production
  script:
    - terraform apply ...
  # Ẩn log sau khi job thành công
  artifacts:
    when: on_failure  # Chỉ lưu artifacts khi fail để debug
```

### Giới Hạn Quyền Token

```yaml
# Chỉ cấp quyền tối thiểu cho CI_JOB_TOKEN
default:
  id_tokens:
    AWS_OIDC_TOKEN:
      aud: sts.amazonaws.com

# Dùng OIDC thay vì long-lived credentials
before_script:
  - |
    export AWS_ROLE_ARN="arn:aws:iam::123456789:role/GitLabCIRole"
    export AWS_WEB_IDENTITY_TOKEN_FILE=/tmp/web-identity-token
    echo "${AWS_OIDC_TOKEN}" > $AWS_WEB_IDENTITY_TOKEN_FILE
    aws sts assume-role-with-web-identity \
      --role-arn $AWS_ROLE_ARN \
      --role-session-name gitlab-ci \
      --web-identity-token file://$AWS_WEB_IDENTITY_TOKEN_FILE \
      --query "Credentials" --output json > /tmp/creds.json
    export AWS_ACCESS_KEY_ID=$(jq -r '.AccessKeyId' /tmp/creds.json)
    export AWS_SECRET_ACCESS_KEY=$(jq -r '.SecretAccessKey' /tmp/creds.json)
    export AWS_SESSION_TOKEN=$(jq -r '.SessionToken' /tmp/creds.json)
```

---

## 📊 GitLab Terraform Integration

### Hiển Thị Terraform State Trong GitLab UI

```
GitLab cung cấp giao diện để xem Terraform state:
Project → Infrastructure → Terraform

Hiển thị:
├── Danh sách tất cả state files
├── Lịch sử thay đổi
├── Tài nguyên đang được quản lý
└── Lock status — Trạng thái khoá
```

### Terraform Merge Request Widget

```yaml
plan:
  stage: plan
  script:
    - terraform plan -out=plan.cache
    # Chuyển đổi plan sang JSON để GitLab đọc được
    - terraform show -json plan.cache > plan.json
  artifacts:
    # GitLab tự động hiển thị Terraform plan trong MR widget
    reports:
      terraform: plan.json
```

---

## ⚡ Tối Ưu Tốc Độ Pipeline

### Parallel Jobs — Chạy Song Song

```yaml
stages:
  - validate-and-lint  # Chạy song song thay vì tuần tự
  - security
  - plan
  - apply

fmt:
  stage: validate-and-lint
  # ...

validate:
  stage: validate-and-lint  # Cùng stage → chạy song song với fmt
  # ...

tflint:
  stage: validate-and-lint  # Cùng stage → chạy song song
  # ...
```

### DAG — Directed Acyclic Graph — Phụ Thuộc Linh Hoạt

```yaml
# Dùng needs: để không phải chờ toàn bộ stage trước
apply:
  stage: apply
  needs:
    - plan          # Chỉ cần plan hoàn thành, không cần các jobs khác
  when: manual
```

---

## 🆚 So Sánh GitHub Actions vs GitLab CI

| Tiêu Chí | GitHub Actions | GitLab CI |
|----------|----------------|-----------|
| **Cú pháp** | `.github/workflows/*.yml` | `.gitlab-ci.yml` (một file) |
| **Reuse** | Reusable Workflows | YAML anchors, `include` |
| **Terraform State** | S3/GCS/Azure Blob | Tích hợp sẵn (HTTP backend) |
| **MR/PR Integration** | Qua GitHub Script | Qua API hoặc native widget |
| **Self-hosted** | GitHub Enterprise | GitLab Self-managed |
| **Security Reports** | Third-party actions | Native Security Dashboard |
| **Approval Gates** | Environment protection | Protected Environments |
| **Phổ biến trong** | Startup, open source | Doanh nghiệp, on-premise |

---

## 📋 Checklist GitLab CI + Terraform

### Pipeline Configuration
- [ ] Stages được định nghĩa rõ ràng theo thứ tự
- [ ] YAML anchors được dùng để tránh lặp lại code
- [ ] Terraform version được pin
- [ ] Cache được cấu hình cho `.terraform/`

### Bảo Mật
- [ ] Sensitive variables được Mask
- [ ] Protected variables chỉ dùng trên protected branches
- [ ] OIDC được cấu hình thay vì long-lived credentials
- [ ] Plan artifacts expire trong thời gian ngắn (1-2 giờ)

### Quy Trình
- [ ] Plan tự động chạy trên MR
- [ ] Apply chỉ chạy trên main branch
- [ ] Manual approval gate cho production
- [ ] Plan output được post vào MR comment hoặc widget

---

## 🎯 Câu Hỏi Phỏng Vấn Về GitLab CI + Terraform

**Q: GitLab Managed Terraform State khác gì với S3 backend?**

A: GitLab Terraform State backend là HTTP backend được GitLab host sẵn. Ưu điểm: không cần tạo S3 bucket riêng, tích hợp với GitLab authentication (dùng CI_JOB_TOKEN), có UI để xem state trong GitLab. Nhược điểm: phụ thuộc vào GitLab infrastructure, khó migrate nếu chuyển platform. S3 backend phổ biến hơn và portable hơn.

**Q: Cách xử lý khi cần apply cho nhiều môi trường từ cùng một pipeline?**

A: Dùng parallel jobs với biến môi trường khác nhau, kết hợp với Protected Environments để kiểm soát ai được phép deploy vào đâu. Pattern phổ biến: develop branch → dev, merge request → staging plan, main branch → staging apply (tự động) → production apply (manual + approval).

---

**Tiếp Theo:** [3-atlantis.md](3-atlantis.md)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
