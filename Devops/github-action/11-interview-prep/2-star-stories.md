# Mẫu Câu Chuyện STAR cho Phỏng Vấn GitHub Actions

> STAR — Situation (Bối Cảnh), Task (Nhiệm Vụ), Action (Hành Động), Result (Kết Quả) — phương pháp trả lời câu hỏi behavioral interview chuẩn mực.

---

## 📖 Hướng Dẫn Sử Dụng

Mỗi câu chuyện bên dưới là **template có thể tùy chỉnh**. Điền vào các placeholder `[...]` bằng thông tin thực tế từ dự án của bạn.

**Nguyên tắc kể chuyện STAR hiệu quả:**
- **Cụ thể:** Dùng con số thực tế (thời gian, phần trăm cải thiện, số lượng)
- **Hành động của bạn:** Dùng "Tôi" không phải "Chúng tôi" — interviewer muốn biết bạn đã làm gì
- **Kết quả đo được:** "Giảm 40% thời gian deploy" tốt hơn "Cải thiện performance"
- **Bài học rút ra:** Bonus điểm nếu đề cập đến điều bạn học được

---

## 📚 10 Câu Chuyện STAR Mẫu

---

### Story 1: Cải Thiện CI Pipeline Quá Chậm

**Câu hỏi trigger:** "Kể về lần bạn tối ưu một hệ thống CI/CD. Bạn đạt được gì?"

---

**Situation (Bối Cảnh):**

Tôi làm việc tại [tên công ty / loại công ty — startup SaaS/enterprise]. Pipeline CI của team tốn **[X] phút** cho mỗi Pull Request. Với team [N] developers commit [Y] lần/ngày, mỗi ngày chúng tôi lãng phí [tính toán] phút chờ đợi. Developers bắt đầu bỏ qua feedback từ CI vì chờ quá lâu, dẫn đến nhiều lỗi lọt qua code review.

**Task (Nhiệm Vụ):**

Nhiệm vụ của tôi là giảm thời gian CI pipeline xuống dưới [target] phút mà không cắt giảm test coverage (Độ Phủ Kiểm Thử). Tôi có 2 tuần để implement và không có budget cho infrastructure mới.

**Action (Hành Động):**

Tôi bắt đầu bằng việc phân tích timing logs của từng step để tìm bottleneck (Điểm Nghẽn):

1. **Phát hiện:** `npm install` chiếm [X] phút vì không có cache. Tôi thêm `actions/cache` với `hashFiles('package-lock.json')` làm cache key → giảm xuống còn [Y] giây khi cache hit.

2. **Phát hiện:** Test suite chạy tuần tự (Sequential) thay vì song song (Parallel). Tôi refactor thành matrix strategy với [N] groups, mỗi group chạy một phần test suite → giảm [X] phút.

3. **Phát hiện:** Build Docker image chạy trên mọi push kể cả doc changes. Tôi thêm `paths` filter để chỉ trigger khi `src/**` hoặc `Dockerfile` thay đổi.

4. **Phát hiện:** Integration tests (Kiểm Thử Tích Hợp) nặng nhất chạy trên mỗi commit. Tôi tách thành scheduled workflow chạy 3 lần/ngày và chỉ bắt buộc với merge vào main.

**Result (Kết Quả):**

- Pipeline giảm từ **[X] phút → [Y] phút** (giảm [Z]%)
- GitHub Actions billing cost giảm **[N]%**
- Developer satisfaction tăng — feedback loop (Vòng Phản Hồi) nhanh hơn khuyến khích commit thường xuyên hơn
- Code coverage thực tế **tăng** vì developers không ngại chờ CI nữa

**Bài học:** Đo lường trước khi optimize (Tối Ưu) — không phải mọi optimization đều cho ROI (Return on Investment — Tỷ Suất Hoàn Vốn) cao như nhau.

---

### Story 2: Xây Dựng CD Pipeline Với Zero-Downtime

**Câu hỏi trigger:** "Bạn đã thiết kế một deployment pipeline từ đầu chưa? Quy trình ra sao?"

---

**Situation (Bối Cảnh):**

Team của tôi có [N] microservices deploy lên [AWS/GCP/Azure]. Deployment process lúc đó là **hoàn toàn thủ công** — developer SSH vào server, chạy script deploy, rồi test bằng tay. Mỗi release mất [X] giờ và thường có downtime [Y] phút.

**Task (Nhiệm Vụ):**

Tôi được giao thiết kế và implement CD pipeline (Đường Ống Phân Phối Liên Tục) tự động hóa toàn bộ quá trình deploy — từ merge PR đến production, với zero downtime và khả năng rollback trong 5 phút.

**Action (Hành Động):**

**Giai đoạn 1 — Thiết kế (1 tuần):**
- Vẽ sơ đồ pipeline với các stages (Giai Đoạn): build → staging → smoke test → approval → production
- Định nghĩa environment protection rules (Quy Tắc Bảo Vệ Môi Trường): production yêu cầu 2 reviewer approvals
- Chọn Blue/Green deployment (Triển Khai Xanh/Xanh Lá) để đảm bảo zero downtime

**Giai đoạn 2 — Implementation (2 tuần):**

```yaml
# Cấu trúc pipeline tôi thiết kế
build → push-image → deploy-staging → e2e-tests → [manual approval] → deploy-production
```

- Implement Docker image build với semantic versioning (Đánh Số Phiên Bản Ngữ Nghĩa)
- Cấu hình OIDC (OpenID Connect) thay thế AWS access keys trong secrets
- Viết smoke tests tự động chạy sau mỗi staging deployment
- Tích hợp Slack notifications (Thông Báo) cho deploy events

**Giai đoạn 3 — Rollout (1 tuần):**
- Deploy cho 1 service pilot, giám sát 2 tuần
- Thu thập feedback từ developers
- Rollout dần cho tất cả [N] services

**Result (Kết Quả):**

- Thời gian release từ [X] giờ → **[Y] phút** (tự động hóa)
- Downtime deployment: từ [X] phút → **0 phút** (Blue/Green)
- Số lần deployment/tuần: từ 1 → [N] (confidence tăng cao)
- Rollback time: **< 5 phút** (switch traffic về Blue)
- Không còn "thứ sáu deploy" syndrome — team deploy thoải mái bất kỳ ngày nào

---

### Story 3: Xử Lý Production Incident Do Deployment

**Câu hỏi trigger:** "Kể về lần deployment gây ra incident. Bạn xử lý như thế nào?"

---

**Situation (Bối Cảnh):**

Lúc [giờ] ngày [thứ/cuối tuần], pipeline CD tự động deploy phiên bản [X.Y.Z] lên production. Trong vòng [N] phút, monitoring (Giám Sát) báo error rate tăng từ 0.1% lên [X]% và response time tăng 5x. Có [N] users đang active bị ảnh hưởng.

**Task (Nhiệm Vụ):**

Với tư cách on-call engineer, tôi phải: (1) restore service nhanh nhất có thể, (2) tìm nguyên nhân gốc rễ (root cause), (3) prevent (Ngăn Chặn) tái diễn.

**Action (Hành Động):**

**T+0 phút — Acknowledge & Triage:**
- Acknowledge alert trong PagerDuty
- Confirm correlation giữa deploy và incident bằng cách so sánh timestamps
- Quyết định rollback ngay thay vì debug trực tiếp (giảm impact trước)

**T+3 phút — Rollback:**
```bash
# Trigger manual rollback workflow
gh workflow run rollback.yml -f version=X.Y.Y -f environment=production
```

**T+8 phút — Service Restored:**
- Rollback hoàn tất, error rate về 0.1%
- Thông báo team qua Slack war room channel

**T+30 phút — Root Cause Analysis:**
- Review diff giữa X.Y.Y và X.Y.Z trong GitHub
- Phát hiện: database query mới không có index → full table scan (Quét Toàn Bảng) → timeout

**T+2 giờ — Fix & Prevention:**
- Tạo hotfix branch, thêm database index
- Thêm query performance test vào CI pipeline
- Cập nhật deployment checklist: "review slow query log sau staging deploy"

**Result (Kết Quả):**

- Downtime: **8 phút** (acceptable theo SLA — Service Level Agreement — Thỏa Thuận Mức Dịch Vụ)
- Rollback thực hiện trong **3 click** nhờ pipeline có sẵn
- Post-mortem document: 5 action items, tất cả completed trong 1 tuần
- Thêm được automated performance regression test vào CI — catch được [N] issues sau đó

**Bài học:** Rollback capability (Khả Năng Quay Lui) phải được test định kỳ, không chỉ xây dựng và để đó. Chúng tôi bắt đầu chaos testing (Kiểm Thử Hỗn Loạn) rollback hàng tháng sau incident này.

---

### Story 4: Migration Sang OIDC Từ Long-lived Credentials

**Câu hỏi trigger:** "Bạn đã cải thiện security của CI/CD pipeline như thế nào?"

---

**Situation (Bối Cảnh):**

Security audit (Kiểm Toán Bảo Mật) phát hiện toàn bộ [N] repositories đang lưu AWS access keys trong GitHub Secrets với expiration (Hết Hạn) không có. Một số keys không được rotate (Xoay Vòng) từ [X] tháng trước. Nếu bị leak, attacker có thể access AWS resources vĩnh viễn cho đến khi key bị revoke thủ công.

**Task (Nhiệm Vụ):**

Migrate tất cả CI/CD workflows sang OIDC (OpenID Connect — Xác Thực Không Cần Credentials Dài Hạn) trong 4 tuần — không break existing deployments và không có downtime.

**Action (Hành Động):**

**Tuần 1 — Inventory & Planning:**
- Audit tất cả [N] repositories: ai đang dùng AWS credentials, dùng cho gì
- Nhóm theo use case: S3 deploy, ECR push, ECS update, Lambda deploy
- Thiết kế IAM Role structure (Cấu Trúc Vai Trò IAM) với least-privilege (Quyền Tối Thiểu)

**Tuần 2 — Infrastructure:**
```hcl
# Terraform tạo OIDC Identity Provider và IAM Roles
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  thumbprint_list = [...]
}

resource "aws_iam_role" "github_actions_deploy" {
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:org/*:ref:refs/heads/main"
        }
      }
    }]
  })
}
```

**Tuần 3 — Migration (Dần Dần):**
- Pilot 2 repos ít critical → validate → migrate thêm 5 repos → validate → migrate tất cả
- Mỗi migration: test trong staging trước, deploy sau
- Giữ old secrets trong 2 tuần trước khi xóa (fallback plan)

**Tuần 4 — Cleanup & Documentation:**
- Xóa tất cả AWS access keys khỏi GitHub Secrets
- Revoke IAM Users dùng cho CI/CD
- Viết documentation cho pattern mới
- Trình bày cho team về cách dùng OIDC

**Result (Kết Quả):**

- **0 long-lived credentials** trong GitHub Secrets sau migration
- Temporary credentials expire (Hết Hạn) sau **15 phút** — blast radius (Bán Kính Phá Hủy) giảm drastically
- Không có incident trong quá trình migration
- Security posture score (Điểm Trạng Thái Bảo Mật) tăng từ [X] → [Y] trong next audit
- **Bài học** cho team về zero-trust cloud access

---

### Story 5: Xây Dựng Platform CI/CD Dùng Chung Cho 20+ Repos

**Câu hỏi trigger:** "Làm thế nào bạn standardize CI/CD across multiple teams?"

---

**Situation (Bối Cảnh):**

Công ty có [N] teams, mỗi team có cách viết CI/CD riêng. Kết quả: [N] workflow implementations khác nhau, security practices không đồng nhất, không ai chịu trách nhiệm khi có vulnerability trong shared workflows. Security team phát hiện 3 repos đang dùng actions không được pin by SHA (Gắn Bằng SHA).

**Task (Nhiệm Vụ):**

Thiết kế và implement Platform CI/CD — một bộ reusable workflows (Workflow Tái Sử Dụng) chuẩn hóa cho toàn tổ chức, với security baked-in và developer experience tốt.

**Action (Hành Động):**

**Tháng 1 — Discovery & Design:**
- Survey [N] teams: pain points, requirements, tech stacks
- Phân loại: 70% Node.js, 20% Python, 10% Java
- Thiết kế `central-ci` repository với reusable workflow library

**Tháng 2 — Build Core Platform:**

```yaml
# central-ci/.github/workflows/ci-node.yml
on:
  workflow_call:
    inputs:
      node-version:
        default: '20'
        type: string
      test-command:
        default: 'npm test'
        type: string
    secrets:
      npm-token:
        required: false

jobs:
  security-scan:             # Bắt buộc chạy cho mọi repo
    uses: ./.github/workflows/security-scan.yml

  lint-test-build:
    needs: security-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a8b7c6d5e4f3   # Pinned SHA
      - uses: actions/setup-node@39370e3970a6
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: ${{ inputs.test-command }}
```

**Tháng 3 — Adoption:**
- Migrate 5 repos pilot với đội ngũ enthusiastic
- Đo metrics (Chỉ Số): adoption time, support tickets, CI time
- Fine-tune dựa trên feedback
- Write migration guide cho các teams còn lại

**Result (Kết Quả):**

- **[N] repos** đã migrate trong 6 tháng
- Security scan **100% coverage** — không còn repo không có SAST (Static Application Security Testing — Kiểm Thử Bảo Mật Ứng Dụng Tĩnh)
- Thời gian setup CI cho project mới: từ [X] giờ → **30 phút**
- Số lượng unique workflow implementations: từ [N] → **3** (Node, Python, Java)
- Developer satisfaction: [X]/10 → [Y]/10 trong survey

---

### Story 6: Debug Workflow Bí Ẩn Chỉ Fail Trong Production

**Câu hỏi trigger:** "Kể về một bug khó debug liên quan đến CI/CD. Bạn tìm ra nguyên nhân thế nào?"

---

**Situation (Bối Cảnh):**

Pipeline production deploy bỗng nhiên bắt đầu fail sau khi team không thay đổi gì trong [N] ngày. Lạ hơn, staging deploy vẫn chạy hoàn toàn bình thường. Error message chỉ là `Error: Process completed with exit code 1` — không có thêm thông tin gì.

**Task (Nhiệm Vụ):**

Tìm nguyên nhân gốc rễ và fix trong [X] giờ — production deploy đang bị blocked.

**Action (Hành Động):**

**Bước 1 — Bật Debug Logging:**
```bash
# Thêm vào Repository Secrets:
ACTIONS_STEP_DEBUG = true
ACTIONS_RUNNER_DEBUG = true
```

Re-run workflow → nhận được detailed logs → thấy lỗi thực sự: AWS credential expired (Hết Hạn).

**Bước 2 — Điều Tra Sâu Hơn:**
- Staging dùng OIDC → credentials tự refresh → OK
- Production vẫn dùng IAM User access key từ [X] tháng trước
- AWS mặc định enforce key rotation (Xoay Vòng Khóa) sau 90 ngày → key bị deactivated tự động

**Bước 3 — Fix:**
- Migrate production workflow sang OIDC ngay
- Validate trong staging trước
- Deploy — fixed!

**Bước 4 — Prevention:**
- Audit tất cả repositories còn dùng long-lived credentials
- Setup Dependabot và scheduled workflow để detect expiring credentials
- Ưu tiên OIDC migration cho toàn tổ chức

**Result (Kết Quả):**

- Pipeline restored sau **[X] giờ**
- Phát hiện thêm [N] repos có cùng vấn đề → proactive fix trước khi bị affect
- Migration sang OIDC hoàn toàn trong [N] tuần sau đó
- **Bài học lớn:** Staging và production nên dùng cùng authentication mechanism

---

### Story 7: Implement GitOps Cho Kubernetes Deployments

**Câu hỏi trigger:** "Bạn đã làm việc với Kubernetes deployment pipeline chưa? Kể chi tiết."

---

**Situation (Bối Cảnh):**

Team vận hành [N] microservices trên Kubernetes. Deployment process: developer push code → CI build image → **developer thủ công SSH vào bastion** (Jump Server — Máy Chủ Nhảy) → chạy `kubectl apply` → verify. Process này:
- Không có audit trail (Nhật Ký Kiểm Toán) rõ ràng
- Dễ xảy ra human error (Lỗi Người Dùng)
- Không thể rollback nhanh
- Blocking với developer

**Task (Nhiệm Vụ):**

Implement GitOps pattern (Mẫu Hoạt Động Git) — mọi thay đổi infrastructure đều qua Git, tự động apply bởi CD pipeline.

**Action (Hành Động):**

**Kiến trúc thiết kế:**

```
app-repo (application code)
    ↓ (CI builds image, tags với git SHA)
k8s-manifests-repo (Kubernetes configs — GitOps repo)
    ↓ (pipeline update image tag)
    ↓ (ArgoCD/Flux tự động apply)
Kubernetes Cluster
```

**Implementation:**

```yaml
# app-repo CI — sau khi build image thành công
- name: Update Kubernetes manifest
  run: |
    git clone https://github.com/org/k8s-manifests
    cd k8s-manifests
    # Cập nhật image tag trong manifest
    sed -i "s|image: myapp:.*|image: myapp:${{ github.sha }}|" \
      apps/myapp/deployment.yaml
    git config user.email "ci@example.com"
    git add .
    git commit -m "Update myapp to ${{ github.sha }}"
    git push
```

- Cấu hình ArgoCD (Công Cụ Triển Khai Liên Tục) watch `k8s-manifests-repo`
- ArgoCD tự động sync khi manifest thay đổi
- Rollback = revert commit trong k8s-manifests-repo

**Result (Kết Quả):**

- **100% audit trail** — mọi deployment có commit trong Git
- Rollback time từ [X] phút → **2 phút** (git revert + pipeline)
- Không còn SSH vào bastion — tăng security posture
- Team velocity tăng — developers deploy tự tin hơn
- Compliance team hài lòng vì có đầy đủ deployment history

---

### Story 8: Reduce GitHub Actions Cost 50% Mà Không Mất Coverage

**Câu hỏi trigger:** "Bạn đã tối ưu chi phí infrastructure như thế nào?"

---

**Situation (Bối Cảnh):**

Nhận email từ finance team: GitHub Actions bill tháng đó là $[X] — tăng [Y]% so với tháng trước mà team size không tăng tương ứng. Cần giải thích và giảm cost.

**Task (Nhiệm Vụ):**

Phân tích nguyên nhân tăng cost và tìm cách giảm ít nhất 30% mà không ảnh hưởng đến quality gates (Cổng Chất Lượng).

**Action (Hành Động):**

**Phân tích:** Dùng GitHub REST API để lấy workflow duration data:

```python
import requests
# Lấy top 10 workflows tốn minutes nhất
runs = requests.get('/repos/org/repo/actions/runs').json()
# Phân tích theo job, runner type, thời gian
```

**Phát hiện:**
1. **45%** của minutes đến từ một scheduled workflow chạy mỗi 15 phút — nhưng thực ra chỉ cần 1 lần/giờ
2. **25%** từ integration tests chạy trên mọi push, kể cả doc-only changes
3. **15%** từ Ubuntu 8-core runners cho jobs chỉ cần 2-core
4. **10%** từ duplicate runs — `on: push` và `on: pull_request` trigger cùng lúc

**Actions:**
1. Scheduled workflow: 15 phút → 60 phút → **-45%** minutes từ source này
2. Path filtering cho integration tests → **-80%** runs cho doc changes
3. Downgrade runner size cho simple jobs → **-75%** cost cho những jobs đó
4. Thêm concurrency group để cancel duplicate runs

**Result (Kết Quả):**

- Total monthly cost: $[X] → $[Y] (**-[N]%**)
- Test coverage: không thay đổi — mọi quality gate vẫn active
- Build time cho critical path: không thay đổi (scheduled jobs không critical)
- Bonus: CI feedback time trên PRs nhanh hơn vì ít jobs hơn chạy cùng lúc

---

### Story 9: Setup Self-hosted Runners Cho Compliance Requirements

**Câu hỏi trigger:** "Khi nào bạn chọn self-hosted runners? Bạn có kinh nghiệm setup không?"

---

**Situation (Bối Cảnh):**

Company ký được hợp đồng với một khách hàng trong lĩnh vực tài chính/y tế. Compliance requirements (Yêu Cầu Tuân Thủ) yêu cầu: source code và build artifacts không được xử lý trên infrastructure của bên thứ ba (third-party). GitHub-hosted runners là Microsoft Azure — không đáp ứng yêu cầu này.

**Task (Nhiệm Vụ):**

Setup self-hosted runners trên on-premise infrastructure của công ty trong [N] tuần — đảm bảo security, scalability (Khả Năng Mở Rộng), và developer experience tương đương GitHub-hosted.

**Action (Hành Động):**

**Architecture decision:** Chọn ARC (Actions Runner Controller — Bộ Điều Khiển Runner Actions) trên internal Kubernetes cluster thay vì VMs tĩnh vì:
- Auto-scaling theo demand
- Ephemeral runners (Runner Tạm Thời) — bị xóa sau mỗi job, không có state
- Dễ update runner version

```yaml
# Helm chart cài ARC
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# Scale set cho project cụ thể
helm install arc-runner-set \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/org/repo" \
  --set minRunners=1 \
  --set maxRunners=10
```

**Security hardening (Tăng Cường Bảo Mật):**
- Network policy (Chính Sách Mạng): runners chỉ được egress ra GitHub API và internal registries
- No persistent storage — ephemeral filesystem
- Runner pods chạy với non-root user
- Separate runner pools cho production vs non-production

**Result (Kết Quả):**

- Đáp ứng hoàn toàn compliance requirements — code không rời khỏi data center
- Performance tương đương GitHub-hosted (vì hardware tốt hơn)
- Auto-scaling từ 1 → [N] runners trong [X] phút khi có spike (Đột Tăng)
- Cost: $[X]/tháng infrastructure thay vì $[Y]/tháng GitHub Actions billing

---

### Story 10: Prevent Supply Chain Attack Trong CI/CD

**Câu hỏi trigger:** "Bạn quan tâm đến supply chain security như thế nào?"

---

**Situation (Bối Cảnh):**

Sau khi đọc về SolarWinds supply chain attack (Tấn Công Chuỗi Cung Ứng) và các incidents liên quan đến compromised GitHub Actions, security team của chúng tôi muốn review và harden toàn bộ workflow dependencies. Audit ban đầu phát hiện:
- [N] repos dùng `actions/checkout@v4` thay vì pinned SHA
- Một số repos dùng actions từ organizations chưa được verify
- Không có automated monitoring cho action updates

**Task (Nhiệm Vụ):**

Implement supply chain security best practices cho tất cả CI/CD workflows trong [N] tuần.

**Action (Hành Động):**

**Bước 1 — Audit & Inventory:**
```bash
# Script tìm tất cả actions dùng tag thay vì SHA
grep -r "uses:" .github/workflows/ | grep -v "@[0-9a-f]\{40\}"
```

Phát hiện [N] workflows cần update.

**Bước 2 — Pin tất cả actions bởi SHA:**
```yaml
# Trước (NGUY HIỂM)
- uses: actions/checkout@v4

# Sau (AN TOÀN)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

Viết script tự động convert tag → SHA cho tất cả workflows.

**Bước 3 — Automated Updates với Dependabot:**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    # Dependabot tự tạo PRs cập nhật SHA khi có version mới
```

**Bước 4 — Dependency Review:**
```yaml
# Required workflow cho toàn tổ chức
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: high
```

**Result (Kết Quả):**

- **100% actions** được pin bởi SHA đầy đủ
- Automated PRs từ Dependabot: team review và approve thay vì miss updates
- Dependency review block [N] PRs với known vulnerable dependencies trong 6 tháng
- Security audit score tăng [X] điểm — passed ISO 27001 certification

---

## 🎯 Template Tùy Chỉnh Nhanh

Dùng template này để viết câu chuyện của riêng bạn:

```markdown
### [Tên Câu Chuyện]

**Câu hỏi trigger:** "[Câu hỏi interviewer hay hỏi]"

**Situation:**
- Công ty/team: [loại, quy mô]
- Vấn đề: [mô tả cụ thể]
- Impact: [ai bị ảnh hưởng, như thế nào]

**Task:**
- Nhiệm vụ của TÔI: [không phải của team]
- Constraints (Ràng Buộc): [thời gian, resources, requirements]
- Success criteria (Tiêu Chí Thành Công): [đo được gì khi xong]

**Action:** (phần quan trọng nhất)
1. [Hành động cụ thể, dùng động từ mạnh: thiết kế, implement, phân tích, refactor...]
2. [Quyết định kỹ thuật + lý do chọn]
3. [Challenge gặp phải và cách giải quyết]

**Result:**
- Metric cứng: [con số cụ thể]
- Business impact: [tiết kiệm thời gian/tiền, tăng reliability...]
- Side effects tích cực: [unexpected wins]
- Bài học: [điều bạn làm khác nếu làm lại]
```

---

## 📝 Checklist Chuẩn Bị Trước Phỏng Vấn

- [ ] Chọn 3–5 câu chuyện phù hợp nhất với vị trí ứng tuyển
- [ ] Tùy chỉnh với số liệu thực tế từ dự án của bạn
- [ ] Luyện kể to thành lời — không phải đọc
- [ ] Đảm bảo mỗi câu chuyện kể trong 3–4 phút (không ngắn hơn 2, không dài hơn 5)
- [ ] Chuẩn bị follow-up sâu hơn cho mỗi câu chuyện (interviewer có thể hỏi thêm)
- [ ] Có câu chuyện về lần thất bại/sai lầm — interviewer thích sự thành thật

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
