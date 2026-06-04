# Atlantis — PR-Based Terraform Automation

> Atlantis — Công Cụ Tự Động Hoá Terraform Dựa Trên Pull Request — cho phép team chạy `terraform plan` và `terraform apply` ngay trong PR comment mà không cần pipeline CI phức tạp.

---

## 🎯 Mục Tiêu

- Hiểu Atlantis hoạt động như thế nào
- Cài đặt và cấu hình Atlantis server
- Viết `atlantis.yaml` cho dự án thực tế
- So sánh Atlantis với các giải pháp khác
- Biết khi nào nên dùng Atlantis

---

## 🤔 Atlantis Là Gì?

Atlantis là một ứng dụng Go — Ngôn Ngữ Lập Trình Go — chạy như server, lắng nghe webhook từ GitHub/GitLab/Bitbucket. Khi có comment đặc biệt trong PR, Atlantis thực thi các lệnh Terraform và đăng kết quả lại vào PR.

```
Developer
    │
    ├─ Tạo PR với thay đổi Terraform
    │         │
    │         ▼
    │   GitHub/GitLab gửi webhook → Atlantis Server
    │         │
    │         ▼ (Atlantis chạy terraform plan)
    │   Atlantis comment vào PR:
    │   "Đã chạy plan. Kết quả: +2 to add, 0 to change, 0 to destroy"
    │
    ├─ Developer review plan
    │
    ├─ Comment: "atlantis apply"
    │         │
    │         ▼
    │   Atlantis chạy terraform apply
    │         │
    │         ▼
    │   Atlantis comment: "Apply thành công. 2 tài nguyên đã được tạo"
    │
    └─ Merge PR
```

---

## 🏗️ Kiến Trúc Atlantis

```
┌──────────────────────────────────────────────────────────────┐
│                     GitHub / GitLab                          │
│  ┌─────────┐                              ┌──────────────┐  │
│  │   PR    │ ──webhook──────────────────► │   Webhook    │  │
│  │comments │ ◄──comments─────────────────│   Handler    │  │
│  └─────────┘                              └──────────────┘  │
└──────────────────────────────────────────────────────────────┘
                                                    │
                                          ┌─────────▼──────────┐
                                          │   Atlantis Server  │
                                          │                    │
                                          │  ┌─────────────┐  │
                                          │  │ Terraform   │  │
                                          │  │  Binary     │  │
                                          │  └─────────────┘  │
                                          │                    │
                                          │  ┌─────────────┐  │
                                          │  │  atlantis   │  │
                                          │  │   .yaml     │  │
                                          │  └─────────────┘  │
                                          └────────┬───────────┘
                                                   │
                        ┌──────────────────────────┼──────────────────────┐
                        │                          │                      │
                  ┌─────▼─────┐           ┌───────▼──────┐    ┌──────────▼──────┐
                  │    AWS    │           │    GCP       │    │    Azure        │
                  │  Backend  │           │   Backend    │    │    Backend      │
                  └───────────┘           └──────────────┘    └─────────────────┘
```

---

## 🚀 Cài Đặt Atlantis

### Chạy Bằng Docker

```yaml
# docker-compose.yml
version: "3.8"

services:
  atlantis:
    image: ghcr.io/runatlantis/atlantis:latest
    ports:
      - "4141:4141"
    environment:
      # GitHub configuration
      ATLANTIS_GH_USER: "atlantis-bot"
      ATLANTIS_GH_TOKEN: "${GITHUB_TOKEN}"
      ATLANTIS_GH_WEBHOOK_SECRET: "${WEBHOOK_SECRET}"
      ATLANTIS_REPO_ALLOWLIST: "github.com/my-org/*"

      # AWS credentials cho Terraform
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"

      # Atlantis config
      ATLANTIS_PORT: "4141"
      ATLANTIS_ATLANTIS_URL: "https://atlantis.example.com"
      ATLANTIS_LOG_LEVEL: "info"

    volumes:
      - atlantis-data:/atlantis
      - ./atlantis.yaml:/etc/atlantis/repos.yaml  # Server-side config

volumes:
  atlantis-data:
```

### Chạy Trên Kubernetes

```yaml
# atlantis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atlantis
  namespace: atlantis
spec:
  replicas: 1  # Phải là 1 — Atlantis không hỗ trợ HA
  selector:
    matchLabels:
      app: atlantis
  template:
    metadata:
      labels:
        app: atlantis
    spec:
      serviceAccountName: atlantis  # Dùng IRSA — IAM Roles for Service Accounts
      containers:
        - name: atlantis
          image: ghcr.io/runatlantis/atlantis:v0.28.0
          ports:
            - containerPort: 4141
          env:
            - name: ATLANTIS_GH_USER
              value: "atlantis-bot"
            - name: ATLANTIS_GH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: atlantis-secrets
                  key: github-token
            - name: ATLANTIS_GH_WEBHOOK_SECRET
              valueFrom:
                secretKeyRef:
                  name: atlantis-secrets
                  key: webhook-secret
            - name: ATLANTIS_REPO_ALLOWLIST
              value: "github.com/my-org/*"
            - name: ATLANTIS_ATLANTIS_URL
              value: "https://atlantis.internal.example.com"
          volumeMounts:
            - name: atlantis-data
              mountPath: /atlantis
            - name: config
              mountPath: /etc/atlantis
      volumes:
        - name: atlantis-data
          persistentVolumeClaim:
            claimName: atlantis-data
        - name: config
          configMap:
            name: atlantis-config

---
apiVersion: v1
kind: Service
metadata:
  name: atlantis
  namespace: atlantis
spec:
  selector:
    app: atlantis
  ports:
    - port: 80
      targetPort: 4141
  type: ClusterIP  # Expose qua Ingress
```

---

## ⚙️ Cấu Hình atlantis.yaml

### File Cấu Hình Của Repo

```yaml
# atlantis.yaml — Đặt ở root của repository
version: 3

# Tự động phát hiện project hay cần khai báo tường minh?
automerge: false  # Không tự merge sau khi apply

# Các projects trong repo
projects:
  # Project đơn giản
  - name: vpc
    dir: infrastructure/vpc
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
        - "../modules/**/*.tf"  # Cũng plan khi module thay đổi
      enabled: true

  # Project với workspace khác nhau cho từng môi trường
  - name: app-dev
    dir: infrastructure/app
    workspace: dev
    terraform_version: v1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "environments/dev.tfvars"
      enabled: true
    apply_requirements:
      - approved     # Phải được approve trước khi apply
      - mergeable    # PR phải mergeable (không có conflict)

  - name: app-staging
    dir: infrastructure/app
    workspace: staging
    terraform_version: v1.7.0
    autoplan:
      enabled: false  # Không tự plan, phải comment thủ công
    apply_requirements:
      - approved
      - mergeable
      - undiverged   # Branch phải up-to-date với base branch

  - name: app-production
    dir: infrastructure/app
    workspace: production
    terraform_version: v1.7.0
    autoplan:
      enabled: false
    apply_requirements:
      - approved
      - mergeable
      - undiverged
    # Workflow tùy chỉnh cho production
    workflow: production-workflow

# Workflows — Quy Trình Tùy Chỉnh
workflows:
  # Workflow mặc định
  default:
    plan:
      steps:
        - init:
            extra_args: ["-input=false"]
        - plan:
            extra_args: ["-input=false", "-var-file=environments/dev.tfvars"]

  # Workflow production với thêm bước kiểm tra
  production-workflow:
    plan:
      steps:
        - run: echo "Chạy plan cho PRODUCTION — kiểm tra kỹ!"
        - init:
            extra_args: ["-input=false"]
        - plan:
            extra_args: ["-input=false", "-var-file=environments/production.tfvars"]
        # Chạy tfsec sau khi plan
        - run: tfsec . --minimum-severity=HIGH
        # Ước tính chi phí
        - run: infracost breakdown --path .
    apply:
      steps:
        - run: echo "Đang apply vào PRODUCTION — $(date)"
        - apply:
            extra_args: ["-input=false"]
        - run: echo "Apply hoàn thành — $(date)"
```

### Server-Side Config — Cấu Hình Phía Server

```yaml
# /etc/atlantis/repos.yaml — Server administrator config
# (Developer không cần biết file này)

repos:
  # Áp dụng cho tất cả repos
  - id: /.*/
    # Cho phép atlantis.yaml override một số settings
    allow_custom_workflows: true
    allowed_overrides:
      - apply_requirements
      - workflow
    # Yêu cầu tối thiểu cho tất cả repos
    apply_requirements:
      - approved

  # Override riêng cho repo production
  - id: github.com/my-org/infrastructure-prod
    apply_requirements:
      - approved
      - mergeable
      - undiverged
    allowed_overrides: []  # Không cho phép override gì cả

# Workflows được phép
workflows:
  default:
    plan:
      steps:
        - init
        - plan
    apply:
      steps:
        - apply
```

---

## 💬 Các Lệnh Atlantis

```
Trong PR comment:

atlantis help               — Hiển thị trợ giúp
atlantis plan               — Chạy plan cho tất cả projects bị ảnh hưởng
atlantis plan -p vpc        — Chạy plan chỉ cho project "vpc"
atlantis plan -w staging    — Chạy plan với workspace "staging"
atlantis apply              — Apply tất cả plans đã được approve
atlantis apply -p vpc       — Apply chỉ project "vpc"
atlantis unlock             — Mở khoá PR (bỏ lock state)

atlantis version            — Hiển thị version Terraform đang dùng
```

### Luồng Làm Việc Thực Tế

```
Bước 1: Tạo PR
        → Atlantis tự động chạy plan (nếu autoplan: true)
        → Comment plan result vào PR

Bước 2: Review Plan
        Developer A đọc plan output
        "Plan: 2 to add, 0 to change, 0 to destroy"
        Approved ✅

Bước 3: Apply
        Developer A comment: "atlantis apply"
        → Atlantis chạy terraform apply
        → Comment kết quả: "Apply complete! Resources: 2 added"

Bước 4: Merge PR
        → Merge vào main
        → Atlantis unlock PR tự động
```

---

## 🔐 Bảo Mật Atlantis

### Vấn Đề: Ai Được Phép Chạy Lệnh?

```yaml
# Chỉ cho phép team members, không phải bất kỳ ai
# Cấu hình trong Server-Side Config:

repos:
  - id: /.*/
    # Chỉ member của org mới được comment lệnh
    allow_draft_prs: false
```

```
GitHub Setting: Settings → Collaborators & teams
→ Chỉ collaborators mới được comment vào PR
→ Atlantis sẽ bỏ qua comment từ người không có quyền
```

### IRSA — IAM Roles for Service Accounts — Trên Kubernetes

```hcl
# Thay vì hardcode AWS credentials, dùng IRSA
resource "aws_iam_role" "atlantis" {
  name = "atlantis"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${local.eks_oidc_issuer}"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.eks_oidc_issuer}:sub" = "system:serviceaccount:atlantis:atlantis"
        }
      }
    }]
  })
}
```

```yaml
# Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: atlantis
  namespace: atlantis
  annotations:
    # Đây là cách IRSA hoạt động
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/atlantis
```

### Webhook Secret — Bí Mật Webhook

```bash
# Tạo webhook secret ngẫu nhiên
openssl rand -hex 32

# Cấu hình trong GitHub:
# Repository → Settings → Webhooks → Add webhook
# Payload URL: https://atlantis.example.com/events
# Content type: application/json
# Secret: <giá trị từ lệnh trên>
# Events: Pull requests, Issue comments, Push
```

---

## 🌍 Atlantis Cho Multi-Repo — Nhiều Repository

### Atlantis Với Monorepo — Một Repository Lớn

```yaml
# atlantis.yaml trong monorepo
version: 3

projects:
  - name: network
    dir: infra/network
    autoplan:
      when_modified:
        - "*.tf"
        - "../../modules/network/**"

  - name: database
    dir: infra/database
    autoplan:
      when_modified:
        - "*.tf"
        - "../../modules/rds/**"
    apply_requirements:
      - approved

  - name: app-servers
    dir: infra/app-servers
    autoplan:
      when_modified:
        - "*.tf"
        - "../../modules/ec2/**"
    apply_requirements:
      - approved
```

### Atlantis Với Nhiều Repos Riêng Lẻ

```yaml
# /etc/atlantis/repos.yaml — Cấu hình cho nhiều repos
repos:
  - id: github.com/my-org/network-infra
    apply_requirements: [approved]

  - id: github.com/my-org/app-infra
    apply_requirements: [approved, mergeable]

  - id: github.com/my-org/production-infra
    apply_requirements: [approved, mergeable, undiverged]
    # Chỉ admin team mới được apply
```

---

## ⚡ So Sánh Atlantis vs GitHub Actions vs Terraform Cloud

| Tiêu Chí | Atlantis | GitHub Actions | Terraform Cloud |
|----------|----------|----------------|-----------------|
| **PR Integration** | ⭐⭐⭐ Xuất sắc | ⭐⭐ Tốt | ⭐⭐ Tốt |
| **Self-hosted** | Bắt buộc | Tuỳ chọn | Không |
| **Chi phí** | Miễn phí (self-host) | Theo phút chạy | Theo user |
| **Độ phức tạp setup** | Trung bình | Thấp | Thấp |
| **Audit Trail** | Qua PR comments | Qua logs | Native |
| **RBAC** | GitHub permissions | GitHub Environments | Granular |
| **Secrets** | Env vars trên server | GitHub Secrets | Workspace vars |
| **Monorepo** | Xuất sắc | Phức tạp | Được |
| **Multi-cloud** | Xuất sắc | Được | Có giới hạn |

---

## ✅ Khi Nào Nên Dùng Atlantis?

### Phù Hợp Khi:

```
✅ Team có hạ tầng phức tạp, nhiều projects trong repo
✅ Muốn plan và apply ngay trong PR comment (không cần merge mới apply)
✅ Có monorepo với nhiều Terraform directories
✅ Muốn full control over execution environment
✅ Team không muốn dùng Terraform Cloud vì chi phí hoặc compliance
✅ Cần apply nhiều môi trường từ một PR
```

### Không Phù Hợp Khi:

```
❌ Team nhỏ, ít thay đổi hạ tầng
❌ Không muốn quản lý thêm infrastructure (Atlantis server)
❌ Không có private network access từ server
❌ Cần audit trail tích hợp sẵn (dùng Terraform Cloud thay)
❌ Team đã có CI/CD system hoạt động tốt
```

---

## 🐛 Troubleshooting Atlantis

### Atlantis Không Nhận Webhook

```bash
# Kiểm tra logs Atlantis
kubectl logs -n atlantis deployment/atlantis -f

# Kiểm tra webhook delivery trong GitHub:
# Repository → Settings → Webhooks → Recent Deliveries
# Xem response code và body

# Lỗi thường gặp:
# 400: Webhook secret không khớp
# 404: URL không đúng
# 503: Atlantis server down
```

### Plan Bị Treo — Hung

```bash
# Mở khoá state thủ công
# Trong PR comment:
atlantis unlock

# Hoặc qua CLI nếu cần
terraform -chdir=infrastructure/app force-unlock <LOCK_ID>
```

### ATLANTIS_REPO_ALLOWLIST Không Match

```
Vấn đề: "Repo not in allowlist"

Kiểm tra:
- Giá trị phải là glob pattern: "github.com/my-org/*"
- Không có trailing slash
- Case sensitive: "GitHub.com" ≠ "github.com"
```

---

## 📋 Checklist Atlantis

### Cài Đặt
- [ ] Atlantis server đang chạy và accessible qua HTTPS
- [ ] Webhook được cấu hình trong GitHub/GitLab
- [ ] Webhook secret khớp giữa GitHub và Atlantis
- [ ] ATLANTIS_REPO_ALLOWLIST chứa repo của bạn

### Cấu Hình
- [ ] `atlantis.yaml` có trong root của repo
- [ ] `apply_requirements` được thiết lập cho production projects
- [ ] `autoplan.when_modified` chứa đúng file patterns
- [ ] Workflow phù hợp với từng môi trường

### Bảo Mật
- [ ] Atlantis chạy với credentials tối thiểu
- [ ] IRSA hoặc instance profile thay vì hardcoded keys
- [ ] Server không expose public nếu dùng private repos
- [ ] Webhook secret được rotate định kỳ

---

## 🎯 Câu Hỏi Phỏng Vấn Về Atlantis

**Q: Atlantis hoạt động thế nào khi có nhiều PR cùng sửa một directory?**

A: Atlantis dùng repository-based locking. Khi PR A đang plan hoặc apply một directory, Atlantis sẽ từ chối plan/apply từ PR B cho cùng directory đó. PR B phải đợi hoặc comment `atlantis unlock` trên PR A để mở khoá. Cơ chế này ngăn tình trạng race condition — xung đột đồng thời — trong Terraform.

**Q: Tại sao Atlantis chỉ nên chạy với 1 replica (không HA)?**

A: Atlantis lưu trạng thái lock và pending operations trong bộ nhớ local của process. Nếu chạy 2 replicas, chúng không share state này nên sẽ không biết replica kia đang lock gì. Giải pháp hiện tại là chạy single replica với PersistentVolume để lưu data. Terraform Cloud giải quyết vấn đề HA tốt hơn.

---

**Tiếp Theo:** [4-terraform-cloud.md](4-terraform-cloud.md)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
