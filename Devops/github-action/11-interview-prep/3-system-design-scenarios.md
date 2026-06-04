# Kịch Bản Thiết Kế CI/CD Pipeline

> System design scenarios (Kịch Bản Thiết Kế Hệ Thống) cho phỏng vấn GitHub Actions — 5 kịch bản thực tế từ startup đến enterprise, kèm hướng dẫn trình bày whiteboard.

---

## 📖 Cách Trả Lời System Design Questions

Khi được hỏi thiết kế hệ thống CI/CD, dùng framework **RADIO**:

```
Requirements (Yêu Cầu)  — Hỏi rõ constraints và goals
Architecture            — Phác thảo high-level design
Deep Dive               — Đi sâu vào từng component
Issues & Trade-offs     — Thảo luận điểm yếu và alternatives
Optimization            — Cải thiện gì nếu có thêm thời gian/resource
```

**Tips:**
- Hỏi làm rõ trước khi vẽ: "Team bao nhiêu người? Tech stack gì? Cloud nào?"
- Vẽ sơ đồ từ trái sang phải: Developer → Source Control → CI → CD → Production
- Thảo luận trade-offs (Đánh Đổi) chủ động — không chờ interviewer hỏi
- Đề xuất metrics để đo success (Thành Công)

---

## 🎯 Kịch Bản 1: Startup SaaS — CI/CD Từ Đầu

**Đề bài:** "Công ty startup có 5 developers, monolith Node.js app, chưa có CI/CD. Thiết kế pipeline từ đầu."

---

### Làm Rõ Yêu Cầu

Trước khi thiết kế, hỏi:
- Deploy lên đâu? (AWS? VPS?)
- Có staging environment chưa?
- SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) yêu cầu gì?
- Team có kinh nghiệm với containers không?
- Budget cho GitHub Actions?

**Giả sử:**
- Deploy lên AWS ECS (Elastic Container Service — Dịch Vụ Container Co Giãn)
- Cần staging + production environments
- Downtime < 5 phút/deploy được chấp nhận ban đầu

---

### Thiết Kế

```
Developer Push
     │
     ▼
┌─────────────┐
│  GitHub PR  │ ──→ [CI Workflow]
└─────────────┘      ├── Checkout
     │                ├── npm install (cached)
     │                ├── Lint (ESLint)
     │                ├── Unit Tests + Coverage
     │                └── Build check
     │
     ▼ (Merge to main)
┌─────────────┐
│  CD Trigger │ ──→ [CD Workflow]
└─────────────┘      ├── Build Docker image
                      ├── Push to ECR (Elastic Container Registry)
                      ├── Deploy to Staging (ECS)
                      ├── Run Smoke Tests
                      ├── [Manual Approval — 1 approver]
                      └── Deploy to Production (ECS)
```

**Workflow files:**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - run: npm run build
```

```yaml
# .github/workflows/cd.yml
name: CD
on:
  push:
    branches: [main]

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    outputs:
      image: ${{ steps.push.outputs.image }}
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1

      - id: push
        uses: aws-actions/amazon-ecr-login@v2
      # ... build và push image

  deploy-staging:
    needs: build-push
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging ECS
        run: |
          aws ecs update-service \
            --cluster staging \
            --service app \
            --force-new-deployment

  smoke-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: |
          for i in {1..5}; do
            curl -f https://staging.example.com/health && break
            sleep 10
          done

  deploy-prod:
    needs: smoke-test
    environment: production   # Requires approval (Yêu Cầu Phê Duyệt)
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service app \
            --force-new-deployment
```

### Trade-offs Cần Thảo Luận

**Chọn ECS thay vì EKS/Lambda:**
- ECS: đơn giản hơn cho team nhỏ, overhead thấp, nhưng ít flexible hơn Kubernetes
- EKS: powerful hơn nhưng learning curve (Đường Cong Học Tập) cao, overhead lớn cho 5 người

**Rolling deployment thay vì Blue/Green:**
- Đơn giản hơn setup ban đầu
- Nếu cần zero-downtime sau này, upgrade lên Blue/Green

**1 approver cho production:**
- Balance (Cân Bằng) giữa speed và safety cho startup
- Enterprise sẽ cần 2+ approvers

### Metrics Thành Công

- Deploy frequency (Tần Suất Triển Khai): >= 5 lần/tuần
- Lead time (Thời Gian Chờ): code merge → production < 30 phút
- Mean Time to Recovery (MTTR — Thời Gian Phục Hồi Trung Bình): < 30 phút với rollback workflow

---

## 🏢 Kịch Bản 2: E-commerce Platform — High Availability

**Đề bài:** "E-commerce platform có 50 developers, 20 microservices, traffic peak (Đỉnh Traffic) Black Friday. Thiết kế CI/CD pipeline."

---

### Làm Rõ Yêu Cầu

- SLA: 99.99% uptime (4 giờ downtime/năm)
- Deploy frequency: 20–30 lần/ngày across all services
- Team: 50 devs trong 8 teams, mỗi team owns 2–3 services
- Traffic: normal 1000 RPS (Requests Per Second — Yêu Cầu Mỗi Giây), peak 50,000 RPS (Black Friday)
- Compliance: PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) vì xử lý thanh toán

---

### Thiết Kế

```
Developer → PR → CI (per service)
                  ├── Lint + Test (matrix: Node 18/20/22)
                  ├── Security Scan (Snyk, SAST)
                  ├── Container Build + Scan (Trivy)
                  └── Performance Test (k6 benchmark)

Merge to main → CD
├── Build & Push to ECR (tagging: SHA + semantic version)
├── Deploy to Staging (Kubernetes Rolling Update)
├── Integration Tests + Contract Tests
├── Load Test (tự động, không cần approval)
│
├── [Auto-deploy to Production nếu không peak hours]
│   OR
└── [Canary Deploy nếu peak hours — 5% traffic trước]
    └── Monitor 30 phút → promote 100% hoặc rollback
```

**Central Workflow Library:**

```
central-ci-platform/
├── .github/workflows/
│   ├── ci-base.yml           # Base CI mọi service kế thừa
│   ├── security-scan.yml     # SAST, dependency review (Required)
│   ├── container-build.yml   # Build, scan, push image
│   ├── deploy-k8s.yml        # Deploy lên Kubernetes
│   └── canary-deploy.yml     # Canary deployment workflow
└── policies/
    └── required-checks.yml   # Định nghĩa checks bắt buộc
```

**Canary Deployment (Triển Khai Canary — Thử Nghiệm Trên Nhóm Nhỏ):**

```yaml
# canary-deploy.yml
jobs:
  canary:
    steps:
      - name: Deploy canary (5% traffic)
        run: |
          kubectl set image deployment/app-canary app=${{ inputs.image }}
          kubectl patch service app --patch '
            spec:
              trafficPolicy:
                canary: 5
          '

      - name: Monitor canary for 30 minutes
        run: |
          # Check error rate mỗi minute trong 30 phút
          for i in {1..30}; do
            ERROR_RATE=$(curl -s prometheus/query?... | jq .data.result[0].value[1])
            if (( $(echo "$ERROR_RATE > 0.01" | bc) )); then
              echo "Error rate too high — rolling back"
              kubectl rollout undo deployment/app-canary
              exit 1
            fi
            sleep 60
          done

      - name: Promote to 100%
        run: kubectl patch service app --patch '{"spec":{"trafficPolicy":{"canary":100}}}'
```

### Trade-offs Chính

**GitOps vs Direct kubectl:**
- GitOps: tốt hơn cho audit trail, ArgoCD auto-sync
- Direct kubectl: simpler setup, lower latency trong pipeline
- **Chọn GitOps** vì PCI-DSS yêu cầu audit trail

**Canary vs Blue/Green:**
- Blue/Green: instant rollback, nhưng cần 2x infrastructure
- Canary: gradual rollout (Phát Hành Dần Dần), phát hiện vấn đề sớm, infrastructure efficient hơn
- **Chọn Canary** cho peak traffic scenarios

**Self-hosted vs GitHub-hosted runners:**
- PCI-DSS compliance → code không được ra ngoài
- **Chọn self-hosted ARC** (Actions Runner Controller) trên internal Kubernetes

---

## 🔐 Kịch Bản 3: Financial Institution — Compliance-heavy

**Đề bài:** "Ngân hàng muốn modernize CI/CD. Requirements: SOC 2, PCI-DSS, mọi thứ phải auditable (Có Thể Kiểm Toán). Thiết kế pipeline."

---

### Requirements Đặc Biệt

- **4-eyes principle** (Nguyên Tắc Bốn Mắt): ít nhất 2 người approve mọi production change
- **Immutable audit log** (Nhật Ký Kiểm Toán Bất Biến): không ai có thể xóa
- **Separation of duties** (Phân Tách Nhiệm Vụ): người build không được là người deploy
- **Change Management**: mọi deploy phải có Change Request (Yêu Cầu Thay Đổi) được approve
- **Code in-house**: không dùng GitHub-hosted runners

---

### Thiết Kế

```
Developer → PR → CI (Self-hosted Runner)
│                 ├── Security Scan (SonarQube, SAST, DAST)
│                 ├── Dependency Check (OWASP)
│                 ├── Unit + Integration Tests
│                 ├── License Compliance Check
│                 └── SBOM Generation (Software Bill of Materials — Danh Sách Thành Phần)
│
│ Code Review: 2 senior reviewers required
│
Merge to main → Artifact Build
│              ├── Signed Docker image (Notary/Cosign)
│              ├── SBOM attached to image
│              └── Stored in private registry
│
│ Change Request: Auto-created in ServiceNow
│ Approval: Change Advisory Board (2+ approvers, 24h window)
│
Production Deploy (After CAB Approval)
├── Deployment record in audit DB
├── Blue/Green strategy
├── Automated smoke tests
└── Alert to compliance team
```

**Signed Commits và Images:**

```yaml
# Yêu cầu signed commits
- name: Verify commit signature
  run: |
    git log --show-signature -1 | grep "Good signature" || \
      (echo "Commit must be signed with GPG" && exit 1)

# Sign Docker image với Cosign
- name: Sign container image
  run: |
    cosign sign --key ${{ secrets.COSIGN_PRIVATE_KEY }} \
      ${{ steps.build.outputs.image }}

# Verify before deploy
- name: Verify image signature
  run: |
    cosign verify --key ${{ secrets.COSIGN_PUBLIC_KEY }} \
      ${{ steps.image.outputs.tag }}
```

**Immutable Audit Trail:**

```yaml
- name: Record deployment to audit log
  run: |
    aws dynamodb put-item \
      --table-name deployment-audit-log \
      --item '{
        "deploymentId": {"S": "${{ github.run_id }}"},
        "timestamp": {"S": "${{ github.event.head_commit.timestamp }}"},
        "commitSha": {"S": "${{ github.sha }}"},
        "approvedBy": {"S": "${{ github.actor }}"},
        "environment": {"S": "production"},
        "imageDigest": {"S": "${{ steps.image.outputs.digest }}"}
      }'
    # DynamoDB table có TTL (Time To Live — Thời Gian Sống) tắt, Point-in-Time Recovery bật
    # → Audit log không thể xóa kể cả admin
```

### Điểm Quan Trọng Khi Thảo Luận

- **Security vs Speed**: Compliance workflows chậm hơn (30–60 phút) nhưng mandatory
- **Automated CAB**: nên automate Change Request creation trong ServiceNow để giảm manual work
- **Separation of duties**: developers không có quyền approve own changes, không có quyền access production secrets

---

## 🌐 Kịch Bản 4: Multi-Cloud — AWS + GCP + On-premise

**Đề bài:** "Công ty dùng AWS cho production, GCP cho AI/ML workloads, on-premise cho legacy. CI/CD thế nào?"

---

### Thách Thức

- 3 clouds/environments với authentication khác nhau
- Consistency (Nhất Quán) trong deployment process
- Secret management (Quản Lý Bí Mật) phức tạp hơn
- Network connectivity giữa environments

---

### Thiết Kế

```
Unified CI (GitHub Actions)
├── OIDC với AWS → deploy ECS, Lambda
├── OIDC với GCP → deploy GKE, Cloud Run
└── VPN Tunnel → deploy on-premise (self-hosted runner ở on-prem network)

Central Secret Management (HashiCorp Vault)
└── Dynamic secrets (Bí Mật Động) cho từng cloud
    ├── AWS: temporary IAM credentials
    ├── GCP: service account tokens
    └── On-prem: internal certificates
```

```yaml
jobs:
  deploy-aws:
    environment: aws-production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: aws ecs update-service ...

  deploy-gcp:
    environment: gcp-ml-platform
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}
      - run: gcloud run deploy ...

  deploy-onprem:
    environment: on-premise
    runs-on: self-hosted    # Runner trong on-prem network
    steps:
      - name: Deploy via Vault Dynamic Credentials
        env:
          VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
          VAULT_TOKEN: ${{ secrets.VAULT_TOKEN }}
        run: |
          # Lấy temporary credentials từ Vault
          CREDS=$(vault read -format=json database/creds/deploy-role)
          ./deploy.sh --user $(echo $CREDS | jq -r .data.username)
```

### Key Decision: Centralized vs Federated CI

| | Centralized | Federated (Phân Tán) |
|---|---|---|
| **Governance** | Dễ enforce policies | Phức tạp hơn |
| **Autonomy** | Teams ít flexibility | Teams có nhiều quyền hơn |
| **Overhead** | Một team maintain platform | Mỗi team tự maintain |
| **Phù hợp** | Enterprise, compliance-heavy | Startup, autonomous teams |

---

## 🔄 Kịch Bản 5: Platform Engineering — Internal Developer Platform

**Đề bài:** "Bạn là platform engineer. Thiết kế Internal Developer Platform (IDP — Nền Tảng Nội Bộ Cho Developer) dựa trên GitHub Actions cho 200 developers."

---

### Goals

- **Self-service:** Developer tạo mới service → CI/CD tự động setup
- **Golden paths (Đường Vàng):** Best practices baked-in, không thể bypass
- **Escape hatches (Lối Thoát Khẩn Cấp):** Khi standard path không đủ, có cách extend
- **Observability:** Central dashboard cho tất cả deployments

---

### Kiến Trúc

```
Developer Experience:
  1. Tạo repo từ template (scaffold service)
  2. CI/CD tự động setup qua GitHub App
  3. Deploy bằng comment trên PR: /deploy staging

Platform Layer:
  central-platform repo
  ├── Reusable workflows (cho mọi service type)
  ├── Organization-level required workflows
  └── Policy-as-code (Open Policy Agent)

Tooling:
  ├── GitHub Apps: auto-setup, deployment commands
  ├── Backstage: service catalog + CI/CD dashboard
  └── Grafana: deployment metrics, DORA metrics
```

**Service Template (Scaffolding — Khung Dàn Giáo):**

```yaml
# Khi developer tạo repo từ template:
# .github/workflows/setup.yml (chạy một lần)
on: create
jobs:
  setup-cicd:
    steps:
      - name: Call Platform API to setup CI/CD
        run: |
          curl -X POST https://platform.internal/setup-cicd \
            -d '{"repo": "${{ github.repository }}", "type": "${{ inputs.service-type }}"}'
          # Platform API tự động:
          # 1. Thêm reusable workflow references
          # 2. Setup environments (staging, production)
          # 3. Add required secrets (từ organization)
          # 4. Configure Dependabot
          # 5. Register service trong Backstage catalog
```

**ChatOps Deployment (Triển Khai Qua Chat):**

```yaml
# GitHub App lắng nghe PR comments
# "/deploy staging" → trigger workflow
on:
  issue_comment:
    types: [created]

jobs:
  parse-command:
    if: contains(github.event.comment.body, '/deploy')
    steps:
      - name: Parse deployment command
        id: parse
        run: |
          COMMENT="${{ github.event.comment.body }}"
          ENV=$(echo $COMMENT | grep -oP '(?<=/deploy )\w+')
          echo "environment=$ENV" >> $GITHUB_OUTPUT

      - name: Trigger deployment
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'deploy.yml',
              ref: context.payload.pull_request.head.ref,
              inputs: { environment: '${{ steps.parse.outputs.environment }}' }
            });
```

**DORA Metrics Dashboard (Bảng Điều Khiển Chỉ Số DORA):**

DORA — DevOps Research and Assessment — 4 chỉ số đo hiệu suất DevOps:

| Metric | Đo Lường | Target (Elite) |
|---|---|---|
| **Deployment Frequency** (Tần Suất Triển Khai) | Số lần deploy/ngày | Multiple per day |
| **Lead Time for Changes** (Thời Gian Chờ) | Commit → Production | < 1 giờ |
| **Change Failure Rate** (Tỷ Lệ Thất Bại) | % deploys gây incident | < 5% |
| **Time to Restore** (Thời Gian Phục Hồi) | Incident → Recovery | < 1 giờ |

```yaml
# Workflow emit metrics sau mỗi deploy
- name: Record DORA metrics
  run: |
    # Lead time = now - first commit timestamp
    LEAD_TIME=$(( $(date +%s) - ${{ steps.commits.outputs.first_commit_ts }} ))
    
    curl -X POST https://metrics.internal/dora \
      -d "{
        \"metric\": \"lead_time\",
        \"value\": $LEAD_TIME,
        \"service\": \"${{ github.repository }}\",
        \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
      }"
```

---

## 📝 Khung Trả Lời Khi Bị Hỏi System Design

### Mở Đầu (2–3 phút)

```
"Trước khi thiết kế, tôi muốn hỏi thêm về requirements:
1. Team có bao nhiêu developers? Bao nhiêu repos?
2. Tech stack chính là gì?
3. Deploy lên đâu — cloud provider nào?
4. SLA yêu cầu gì về uptime và deployment frequency?
5. Có compliance requirements gì không?"
```

### Vẽ High-level (5 phút)

```
Bắt đầu với: Developer → Code → Build → Test → Deploy
Sau đó layer thêm: Security, Monitoring, Rollback
```

### Deep Dive (10–15 phút)

Tập trung vào 2–3 components phức tạp nhất:
- Authentication (OIDC vs PAT)
- Deployment strategy (Rolling, Blue/Green, Canary)
- Secret management

### Trade-offs (5 phút)

```
"Với thiết kế này, trade-offs là:
- Complexity (Độ Phức Tạp): X đơn giản hơn nhưng kém flexible hơn Y
- Cost (Chi Phí): A đắt hơn B nhưng an toàn hơn
- Nếu scale lên 10x, điều tôi thay đổi đầu tiên là..."
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
