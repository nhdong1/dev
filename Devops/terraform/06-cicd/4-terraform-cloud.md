# Terraform Cloud / HCP Terraform

> Terraform Cloud (hiện đổi tên thành HCP Terraform — HashiCorp Cloud Platform Terraform) là nền tảng managed — Được Quản Lý Hoàn Toàn — cung cấp remote execution, state management, policy enforcement, và collaboration cho Terraform.

---

## 🎯 Mục Tiêu

- Hiểu sự khác nhau giữa Terraform CLI, Terraform Cloud, và Terraform Enterprise
- Cấu hình workspace và runs trong Terraform Cloud
- Thiết lập VCS — Version Control System — integration
- Hiểu Sentinel — Policy as Code — Chính Sách Dưới Dạng Mã
- Biết khi nào nên dùng Terraform Cloud thay vì self-hosted CI

---

## 🏗️ Các Sản Phẩm HashiCorp Terraform

```
┌─────────────────────────────────────────────────────────────┐
│  Terraform CLI (Open Source)                               │
│  • Miễn phí hoàn toàn                                      │
│  • Chạy trên máy local hoặc CI                             │
│  • Không có UI, cộng tác phải tự xây dựng                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  HCP Terraform / Terraform Cloud (SaaS)                    │
│  • Free tier: 500 resource-hours/tháng                     │
│  • Managed backend, remote execution, VCS integration      │
│  • Team collaboration, RBAC, audit logs                    │
│  • Sentinel policy (paid tiers)                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Terraform Enterprise (Self-hosted)                        │
│  • Cài đặt on-premise hoặc trong VPC riêng                │
│  • Full features của Terraform Cloud                       │
│  • Compliance: air-gapped environments                     │
│  • Giá cao, dành cho doanh nghiệp lớn                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### Organization — Tổ Chức

```
Organization (my-company)
├── Workspace: production-vpc
├── Workspace: staging-vpc
├── Workspace: dev-database
├── Project: Team Alpha
│   ├── Workspace: alpha-prod
│   └── Workspace: alpha-staging
└── Project: Team Beta
    └── Workspace: beta-prod
```

### Workspace — Không Gian Làm Việc

```
Mỗi Workspace trong Terraform Cloud tương ứng với một:
├── Terraform state file độc lập
├── Bộ variables riêng
├── Run history — Lịch sử thực thi
├── Access controls riêng
└── Notifications riêng

Khác với Terraform CLI workspace:
CLI workspace = switch state file
TF Cloud workspace = unit of isolation hoàn toàn
```

### Run — Lần Thực Thi

```
Types of Runs — Loại thực thi:
├── Speculative Plan: plan không apply, dùng cho PR preview
├── Plan-and-Apply: plan → (approve) → apply
└── Destroy: plan destroy → (approve) → destroy

Run States — Trạng thái thực thi:
Pending → Planning → Planned → Cost Estimated
→ Policy Check → Planned and Finished
→ (Manual Confirm) → Applying → Applied
```

---

## ⚙️ Cấu Hình Backend Terraform Cloud

### Cách 1: Backend Block Trong Code

```hcl
# backend.tf
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      # Option 1: Workspace cụ thể
      name = "production-app"

      # Option 2: Tags — tất cả workspaces có tag này
      # tags = ["app", "production"]
    }
  }

  required_version = ">= 1.1.0"
}
```

### Cách 2: Biến Môi Trường

```bash
# Không cần sửa code, set qua env vars
export TF_CLOUD_ORGANIZATION="my-company"
export TF_WORKSPACE="production-app"

terraform init
terraform plan
```

### Authentication — Xác Thực

```bash
# Cách 1: CLI login (interactive)
terraform login
# → Mở browser, tạo API token, lưu vào ~/.terraform.d/credentials.tfrc.json

# Cách 2: Token trong config file (cho CI)
cat > ~/.terraform.d/credentials.tfrc.json << 'EOF'
{
  "credentials": {
    "app.terraform.io": {
      "token": "${TF_CLOUD_TOKEN}"
    }
  }
}
EOF

# Cách 3: Biến môi trường
export TF_TOKEN_app_terraform_io="your-api-token"
```

---

## 🔗 VCS Integration — Tích Hợp Với Version Control

### Kết Nối GitHub/GitLab

```
Terraform Cloud UI:
Organization Settings → Version Control
→ Add a VCS Provider
→ Chọn: GitHub.com / GitLab.com / Bitbucket

Sau khi kết nối, tạo workspace với VCS:
Workspace → Create Workspace
→ Version Control Workflow
→ Chọn repository
→ Cấu hình:
   Working Directory: terraform/app/
   Terraform Version: 1.7.0
   Auto Apply: Off (recommend cho production)
   VCS Branch: main
```

### VCS Workflow — Quy Trình Làm Việc Với VCS

```
Git Repository
      │
      ├─ Pull Request
      │         │
      │         ▼
      │   Terraform Cloud tự động chạy:
      │   Speculative Plan — Plan chỉ để preview
      │         │
      │         ▼
      │   GitHub PR: "Terraform plan has been run"
      │   Link đến detailed plan trong TF Cloud UI
      │
      └─ Merge to main
                │
                ▼
          Terraform Cloud tự động chạy:
          Plan-and-Apply run
                │
                ▼
          Nếu Auto Apply: Off → Cần click Confirm
          Nếu Auto Apply: On  → Apply tự động
```

---

## 📊 Remote Execution — Thực Thi Từ Xa

### Execution Modes — Chế Độ Thực Thi

```
1. Remote (mặc định):
   • Terraform code upload lên TF Cloud
   • Plan và apply chạy trên TF Cloud servers
   • Không cần credentials trên máy local
   • State lưu trong TF Cloud

2. Local:
   • Plan và apply chạy trên máy của bạn
   • Chỉ state và variables lưu trên TF Cloud
   • Phù hợp khi cần truy cập private network

3. Agent (Enterprise/Team+ plan):
   • Terraform Cloud Agents chạy trong network của bạn
   • Cho phép kết nối đến on-premise resources
   • Phù hợp cho hybrid cloud environments
```

### Agents — Chạy Trong Network Riêng

```bash
# Cài đặt Terraform Cloud Agent
docker run -d \
  -e TFC_AGENT_TOKEN="your-agent-token" \
  -e TFC_AGENT_NAME="prod-agent-1" \
  hashicorp/tfc-agent:latest

# Hoặc trên Kubernetes
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tfc-agent
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tfc-agent
  template:
    spec:
      containers:
        - name: agent
          image: hashicorp/tfc-agent:latest
          env:
            - name: TFC_AGENT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tfc-agent-secrets
                  key: token
            - name: TFC_AGENT_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
EOF
```

---

## 🔐 Variables — Biến Trong Terraform Cloud

### Loại Biến

```
Workspace Variables:
├── Terraform Variables (terraform.tfvars equivalent)
│   ├── Ví dụ: environment = "production"
│   └── Sensitive: Ẩn sau khi save, không thể đọc lại
│
└── Environment Variables (shell env vars)
    ├── Ví dụ: AWS_ACCESS_KEY_ID
    └── Sensitive: Tương tự, ẩn sau khi save

Variable Sets — Tập hợp biến dùng chung:
├── Tạo một lần, áp dụng cho nhiều workspaces
├── Ví dụ: "AWS Production Credentials"
│         → Áp dụng cho tất cả production workspaces
└── Quản lý tập trung, thay đổi một chỗ
```

### Cấu Hình Biến Qua API

```bash
# Thêm variable qua Terraform Cloud API
curl \
  --header "Authorization: Bearer $TF_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{
    "data": {
      "type": "vars",
      "attributes": {
        "key": "db_password",
        "value": "super-secret-password",
        "sensitive": true,
        "category": "terraform",
        "description": "Database password for production"
      }
    }
  }' \
  "https://app.terraform.io/api/v2/workspaces/${WORKSPACE_ID}/vars"
```

---

## 🛡️ Sentinel — Policy as Code

> Sentinel — Chính Sách Dưới Dạng Mã — cho phép định nghĩa policy enforcement — Thực Thi Chính Sách — tự động trước khi apply Terraform.

### Vị Trí Trong Run Pipeline

```
Plan → Cost Estimate → Sentinel Policy Check → Apply
                              │
                    ┌─────────┴──────────┐
                    │                    │
              Policy Pass          Policy Fail
                    │                    │
                Apply             Block Apply (hoặc Advisory)
```

### Ví Dụ Sentinel Policy

```python
# Policy: Không cho phép tạo S3 bucket public
# policies/s3-no-public-access.sentinel

import "tfplan/v2" as tfplan

# Lấy tất cả S3 bucket resources trong plan
all_s3_buckets = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket" and
  rc.mode is "managed" and
  (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# Kiểm tra từng bucket
no_public_buckets = rule {
  all all_s3_buckets as _, bucket {
    # ACL không được là "public-read" hoặc "public-read-write"
    bucket.change.after.acl not in ["public-read", "public-read-write"]
  }
}

# Policy fail nếu có bucket public
main = rule {
  no_public_buckets
}
```

```python
# Policy: Tất cả EC2 instance phải có tag "owner"
# policies/ec2-require-owner-tag.sentinel

import "tfplan/v2" as tfplan

all_ec2_instances = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.mode is "managed" and
  rc.change.actions contains "create"
}

instances_have_owner_tag = rule {
  all all_ec2_instances as _, instance {
    instance.change.after.tags contains "owner" and
    instance.change.after.tags.owner is not null and
    instance.change.after.tags.owner is not ""
  }
}

main = rule {
  instances_have_owner_tag
}
```

### Cấu Hình Policy Set

```hcl
# Trong Terraform Cloud UI:
# Organization → Policy Sets → Create Policy Set

# Hoặc qua code trong repository:
# sentinel.hcl (đặt ở root của policy repo)

policy "s3-no-public-access" {
  source            = "./policies/s3-no-public-access.sentinel"
  enforcement_level = "hard-mandatory"  # Block nếu fail
}

policy "ec2-require-owner-tag" {
  source            = "./policies/ec2-require-owner-tag.sentinel"
  enforcement_level = "soft-mandatory"  # Có thể override với lý do
}

policy "cost-limit" {
  source            = "./policies/cost-limit.sentinel"
  enforcement_level = "advisory"  # Chỉ cảnh báo, không block
}
```

---

## 💰 Run Tasks — Tác Vụ Bên Ngoài Trong Run

> Run Tasks — Tác Vụ Tích Hợp Trong Run — cho phép tích hợp với công cụ bên ngoài (Infracost, Snyk, Bridgecrew) mà không cần Sentinel.

```
Plan → Run Tasks → (optional: Sentinel) → Apply
          │
   ┌──────┴──────────────────────────┐
   │                                 │
Infracost                          Snyk
(Cost estimation)            (Security scan)
```

### Cấu Hình Infracost Run Task

```bash
# 1. Đăng ký Infracost tại Terraform Cloud:
#    Organization → Settings → Integrations → Run Tasks
#    → Add Run Task: Infracost

# 2. Gắn vào workspace:
#    Workspace → Settings → Run Tasks
#    → Add: Infracost
#    → Enforcement: Advisory (chỉ cảnh báo)
#    → Stage: Pre-plan hoặc Post-plan
```

---

## 🔗 API-Driven Workflows — Tích Hợp Qua API

### Trigger Run Từ CI Pipeline

```bash
# Tạo run mới qua API
create_run() {
  local workspace_id=$1
  local message=$2

  curl \
    --header "Authorization: Bearer ${TF_TOKEN}" \
    --header "Content-Type: application/vnd.api+json" \
    --request POST \
    --data "{
      \"data\": {
        \"attributes\": {
          \"is-destroy\": false,
          \"message\": \"${message}\"
        },
        \"type\": \"runs\",
        \"relationships\": {
          \"workspace\": {
            \"data\": {
              \"type\": \"workspaces\",
              \"id\": \"${workspace_id}\"
            }
          }
        }
      }
    }" \
    "https://app.terraform.io/api/v2/runs"
}

# Theo dõi trạng thái run
wait_for_run() {
  local run_id=$1
  local status

  while true; do
    status=$(curl -s \
      --header "Authorization: Bearer ${TF_TOKEN}" \
      "https://app.terraform.io/api/v2/runs/${run_id}" \
      | jq -r '.data.attributes.status')

    echo "Run status: ${status}"

    case $status in
      "applied") echo "Run thành công!"; return 0 ;;
      "errored") echo "Run thất bại!"; return 1 ;;
      "discarded") echo "Run bị hủy!"; return 1 ;;
      *) sleep 10 ;;
    esac
  done
}
```

---

## 🏢 Terraform Cloud Pricing Tiers

```
Free:
├── 500 resource-managed hours/tháng
├── 1 concurrent run
├── Unlimited workspaces
└── Community support

Plus (Team Plan):
├── Unlimited resource hours
├── 3+ concurrent runs
├── Team access management
├── Sentinel (soft-mandatory)
├── SSO — Single Sign-On
└── $20/user/tháng

Business:
├── Audit logging đầy đủ
├── Self-hosted agents
├── Sentinel hard-mandatory
├── Private module registry
├── Custom concurrency
└── Pricing theo agreement
```

---

## ⚡ Terraform Cloud vs Self-Hosted CI

### Dùng Terraform Cloud Khi:

```
✅ Team muốn managed solution, không muốn quản lý CI infrastructure
✅ Cần Policy as Code (Sentinel) tích hợp sẵn
✅ Cần audit trail đầy đủ mà không tự xây dựng
✅ Team nhỏ đến trung bình (< 50 người)
✅ Không có yêu cầu compliance về data locality
✅ Muốn Terraform state và execution cùng một platform
```

### Dùng Self-Hosted CI Khi:

```
✅ Có yêu cầu airgap — Cô lập mạng hoàn toàn
✅ Cần tích hợp với hệ thống nội bộ phức tạp
✅ Đã có CI/CD platform (GitHub Actions, GitLab) hoạt động tốt
✅ Chi phí Terraform Cloud vượt quá budget
✅ Cần full customization của pipeline logic
✅ Compliance yêu cầu không dùng third-party SaaS
```

---

## 📋 Checklist Terraform Cloud

### Setup Cơ Bản
- [ ] Organization đã được tạo
- [ ] VCS provider đã kết nối
- [ ] API token đã được tạo và lưu an toàn
- [ ] Workspace đã được tạo và kết nối với VCS

### Biến & Credentials
- [ ] AWS credentials được lưu dưới dạng environment variables sensitive
- [ ] Variable Sets cho shared credentials đã được cấu hình
- [ ] Sensitive variables không thể đọc lại từ UI

### Workflow
- [ ] Auto Apply được tắt cho production
- [ ] Speculative plans chạy trên PRs
- [ ] Notifications đã được cấu hình

### Policy
- [ ] Sentinel policies đã được viết và test
- [ ] Policy sets gắn với đúng workspaces
- [ ] Enforcement levels phù hợp (hard/soft/advisory)

---

## 🎯 Câu Hỏi Phỏng Vấn Về Terraform Cloud

**Q: Sự khác nhau giữa Workspace trong Terraform CLI và Terraform Cloud?**

A: Terraform CLI workspace chỉ là cơ chế switch state file trong cùng một backend, thường dùng trong cùng codebase. Terraform Cloud workspace là unit of isolation hoàn toàn: mỗi workspace có state riêng, variables riêng, access controls riêng, run history riêng, và settings riêng. Một workspace TF Cloud thường tương ứng với một môi trường (dev, staging, prod) hoặc một component hạ tầng.

**Q: Sentinel là gì và tại sao quan trọng?**

A: Sentinel là framework Policy as Code của HashiCorp, cho phép định nghĩa policy bắt buộc TRƯỚC khi Terraform apply. Ví dụ: "không cho phép tạo S3 bucket public", "EC2 instance phải có owner tag", "chi phí tháng không vượt $10,000". Policy được enforce ở tầng platform, không phụ thuộc vào developer có làm đúng hay không. Quan trọng cho compliance và governance trong tổ chức lớn.

**Q: Khi nào dùng Terraform Cloud Agents?**

A: Khi cần Terraform Cloud (managed, có UI, audit) nhưng hạ tầng nằm trong private network (on-premise, VPC không có internet). Agents là process nhỏ chạy trong network của bạn, nhận job từ Terraform Cloud và thực thi locally, gửi kết quả về TF Cloud. Đây là giải pháp hybrid: quản lý bởi TF Cloud, thực thi trong network riêng.

---

**Tiếp Theo:** [5-rollback-strategy.md](5-rollback-strategy.md)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
