# Marketplace Guide — Hướng Dẫn Sử Dụng GitHub Actions Marketplace

> Chọn lọc, đánh giá và sử dụng actions từ GitHub Marketplace an toàn — tránh supply chain attacks (tấn công chuỗi cung ứng) và dependency rot (phụ thuộc lỗi thời).

## 📚 Mục Lục

1. [Tổng Quan Marketplace](#tổng-quan-marketplace)
2. [Tiêu Chí Đánh Giá Action](#tiêu-chí-đánh-giá-action)
3. [Cách Pin Actions — Cố Định Phiên Bản](#cách-pin-actions)
4. [Tự Động Cập Nhật Với Dependabot](#tự-động-cập-nhật-với-dependabot)
5. [Khi Nào Fork Một Action](#khi-nào-fork-một-action)
6. [Actions Phổ Biến & Tin Cậy](#actions-phổ-biến--tin-cậy)
7. [Actions Nên Tránh](#actions-nên-tránh)
8. [Chính Sách Enterprise](#chính-sách-enterprise)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Marketplace

**GitHub Actions Marketplace** là nơi tập hợp hàng chục nghìn actions do cộng đồng và các tổ chức publish — có thể dùng trực tiếp trong workflow mà không cần viết code từ đầu.

### Rủi Ro Khi Dùng Third-party Actions (Actions Bên Thứ Ba)

```
Third-party action chạy với:
├── Quyền truy cập toàn bộ workspace (source code)
├── GITHUB_TOKEN (có thể đọc/ghi repo nếu permissions đủ)
├── Tất cả environment variables và secrets được inject
└── Khả năng chạy bất kỳ lệnh nào trên runner
```

**Ví dụ supply chain attack thực tế:**
- Action `v1` an toàn → maintainer account bị hack → `v1` tag bị move sang commit độc hại
- Action bị abandon, bán cho kẻ xấu, inject malicious code trong update mới

---

## Tiêu Chí Đánh Giá Action

### Checklist Nhanh Trước Khi Dùng

```
✅ Verified creator hoặc trusted organization?
✅ Stars ≥ 500 và nhiều forks?
✅ Commit gần đây (< 6 tháng)?
✅ Issues được respond không?
✅ License phù hợp (MIT, Apache 2.0)?
✅ action.yml rõ ràng về permissions?
✅ Source code readable — không obfuscated?
✅ Không request permissions không cần thiết?
```

### Mức Độ Tin Cậy

| Mức Độ | Điều Kiện | Ví Dụ |
|---|---|---|
| **Cao nhất** | `actions/` org (GitHub official) | `actions/checkout`, `actions/setup-node` |
| **Cao** | Verified creator (badge xanh) | `aws-actions/`, `google-github-actions/`, `azure/` |
| **Trung bình** | Stars > 1000, maintained, license rõ | `docker/build-push-action` |
| **Thận trọng** | Stars ít, maintainer không quen thuộc | Bất kỳ action < 100 stars |
| **Tránh** | Fork không chính thống, no license, code khó đọc | — |

### Cách Đọc action.yml Để Đánh Giá

```yaml
# Dấu hiệu action ĐÁNG TIN CẬY:
runs:
  using: 'node20'
  main: 'dist/index.js'        # ✅ File bundled rõ ràng

# Dấu hiệu CẦN CẨN THẬN:
runs:
  using: 'node20'
  main: 'index.js'             # ⚠️ Không bundle — phụ thuộc vào node_modules commit

# Dấu hiệu NGUY HIỂM:
runs:
  using: 'node20'
  main: 'src/obfuscated_main.js'  # ❌ Code bị obfuscate — không đọc được
```

---

## Cách Pin Actions

### Tại Sao Phải Pin?

```yaml
# ❌ Nguy hiểm — tag có thể bị thay đổi bất cứ lúc nào
- uses: some-action/deploy@v2

# ✅ An toàn hơn — tag cụ thể (nhưng vẫn có thể bị move)
- uses: some-action/deploy@v2.3.1

# ✅ An toàn nhất — SHA commit không bao giờ thay đổi
- uses: some-action/deploy@a1b2c3d4e5f6789012345678901234567890abcd
```

### Lấy SHA Của Một Action

```bash
# Cách 1: Xem trên GitHub — commit history của repo action
# Vào github.com/some-action/deploy → Commits → Copy SHA

# Cách 2: Dùng GitHub CLI
gh api repos/actions/checkout/git/ref/heads/main \
  --jq '.object.sha'

# Cách 3: Dùng git
git ls-remote https://github.com/actions/checkout refs/tags/v4
# Kết quả: abc123...   refs/tags/v4
```

### Cân Bằng Giữa Bảo Mật và Maintainability

```yaml
jobs:
  ci:
    steps:
      # GitHub official actions — tin tưởng cao, dùng major tag là đủ
      - uses: actions/checkout@v4            # OK cho actions chính thức
      - uses: actions/setup-node@v4          # OK cho actions chính thức

      # Third-party critical actions — nên pin theo SHA
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502  # v4.0.2

      # Third-party utility actions — pin theo tag cụ thể là tối thiểu
      - uses: softprops/action-gh-release@v2.0.4

      # Thêm comment để biết phiên bản gốc
      - uses: some-action/tool@a1b2c3d4  # v3.2.1
```

### Cách Thêm SHA Tự Động

```yaml
# Dùng tool: pin-github-action (https://github.com/mheap/pin-github-action)
npx pin-github-action .github/workflows/ci.yml

# Kết quả tự động thêm SHA:
# uses: actions/checkout@v4
# →
# uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

---

## Tự Động Cập Nhật Với Dependabot

Dependabot — Công Cụ Tự Động Cập Nhật Dependencies — có thể tự động tạo PR khi có action version mới:

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Tự động cập nhật GitHub Actions
  - package-ecosystem: 'github-actions'
    directory: '/'                           # Root directory
    schedule:
      interval: 'weekly'                     # Tần suất kiểm tra: daily | weekly | monthly
      day: 'monday'
      time: '09:00'
      timezone: 'Asia/Ho_Chi_Minh'
    open-pull-requests-limit: 5              # Tối đa 5 PR mở cùng lúc
    reviewers:
      - 'platform-team'
    labels:
      - 'dependencies'
      - 'github-actions'
    commit-message:
      prefix: 'chore(actions)'
    ignore:
      # Bỏ qua major version bumps cho actions nhất định (review thủ công)
      - dependency-name: 'aws-actions/configure-aws-credentials'
        update-types: ['version-update:semver-major']
```

### Workflow Tự Động Merge Dependabot PRs

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Auto-merge Dependabot PRs

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'

    steps:
      - name: Fetch Dependabot metadata
        id: meta
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Auto-merge patch and minor updates
        # Tự động merge chỉ patch và minor updates — major phải review thủ công
        if: |
          steps.meta.outputs.update-type == 'version-update:semver-patch' ||
          steps.meta.outputs.update-type == 'version-update:semver-minor'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Khi Nào Fork Một Action

### Nên Fork Khi:

1. **Action critical nhưng không được maintain tốt** — maintainer không active, issues không được respond
2. **Cần customization** — action gần đúng nhưng cần thêm/bớt tính năng
3. **Security concern** — phát hiện vulnerability, maintainer không fix
4. **Enterprise compliance** — cần code review nội bộ trước khi dùng
5. **Action bị archive/deprecated** — không còn update

### Quy Trình Fork An Toàn

```bash
# 1. Fork repo action về organization
# github.com/original/action → github.com/myorg/action

# 2. Review toàn bộ source code
git clone https://github.com/myorg/action
# Đọc kỹ action.yml, Dockerfile/index.js, dependencies

# 3. Rebuild bundle (với JS action)
npm ci && npm run build

# 4. Tạo tag stable trong fork
git tag v1.0.0-myorg
git push origin v1.0.0-myorg

# 5. Cập nhật workflow để dùng fork
# uses: original/action@v1
# →
# uses: myorg/action@v1.0.0-myorg

# 6. Setup process review updates từ upstream
git remote add upstream https://github.com/original/action
git fetch upstream
# Review changes trước khi merge vào fork
```

### Template Đánh Giá Action (Fork Review Checklist)

```markdown
## Action Fork Review: [action-name]

### Tại Sao Fork?
- [ ] Original không maintained
- [ ] Cần customization: [mô tả]
- [ ] Security concern: [chi tiết]
- [ ] Compliance requirement: [chi tiết]

### Security Review
- [ ] Đọc toàn bộ action.yml
- [ ] Đọc entry point (index.js / entrypoint.sh)
- [ ] Kiểm tra tất cả npm dependencies (npm audit)
- [ ] Xác nhận không có hardcoded credentials
- [ ] Verify permissions không quá rộng

### Dependencies
- [ ] Liệt kê tất cả third-party dependencies
- [ ] Kiểm tra license compatibility

### Test Plan
- [ ] Test trong dry-run environment trước
- [ ] Test với edge cases (empty inputs, missing secrets)

### Maintenance Plan
- [ ] Ai chịu trách nhiệm update fork?
- [ ] Tần suất sync với upstream?
- [ ] Khi nào bỏ fork về dùng original?
```

---

## Actions Phổ Biến & Tin Cậy

### GitHub Official (`actions/` organization)

| Action | Mục Đích | Ví Dụ Dùng |
|---|---|---|
| `actions/checkout` | Checkout source code | Mọi workflow |
| `actions/setup-node` | Cài Node.js | Node.js projects |
| `actions/setup-python` | Cài Python | Python projects |
| `actions/setup-java` | Cài Java + Maven/Gradle | Java projects |
| `actions/setup-go` | Cài Go | Go projects |
| `actions/cache` | Cache dependencies | Mọi workflow cần tối ưu |
| `actions/upload-artifact` | Upload build artifacts | CI/CD pipeline |
| `actions/download-artifact` | Download artifacts | Multi-job workflows |
| `actions/create-release` | Tạo GitHub Release | Release pipeline |
| `actions/github-script` | Chạy JavaScript gọi GitHub API | Automation |

### Cloud Provider Official

| Action | Provider | Mục Đích |
|---|---|---|
| `aws-actions/configure-aws-credentials` | AWS | OIDC auth với AWS |
| `aws-actions/amazon-ecr-login` | AWS | Login ECR |
| `google-github-actions/auth` | GCP | OIDC auth với GCP |
| `google-github-actions/deploy-cloudrun` | GCP | Deploy Cloud Run |
| `azure/login` | Azure | Login Azure |
| `azure/aks-set-context` | Azure | Configure AKS kubectl |

### Community Actions Phổ Biến & Tin Cậy

| Action | Tác Giả | Mục Đích |
|---|---|---|
| `docker/build-push-action` | docker | Build & push Docker image |
| `docker/login-action` | docker | Login container registry |
| `docker/setup-buildx-action` | docker | Setup Docker Buildx |
| `docker/metadata-action` | docker | Tạo Docker image tags |
| `helm/kind-action` | helm | Setup kind cluster để test |
| `hashicorp/setup-terraform` | hashicorp | Cài Terraform |
| `hashicorp/vault-action` | hashicorp | Lấy secrets từ Vault |
| `slackapi/slack-github-action` | slackapi | Gửi Slack notifications |
| `peter-evans/create-pull-request` | peter-evans | Tự động tạo PR |
| `release-drafter/release-drafter` | release-drafter | Draft release notes |

---

## Actions Nên Tránh

### Dấu Hiệu Nhận Biết Action Không An Toàn

```yaml
# ❌ Action request write permissions không cần thiết
- uses: suspicious/action@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}   # Tại sao cần token nếu chỉ build?

# ❌ Action dùng curl/wget để tải script từ internet và execute
- name: Install tool
  run: curl https://get.example.com | bash   # KHÔNG làm thế này

# ❌ Action upload code/data lên external server không rõ lý do
# (Xem source code để phát hiện)

# ❌ Action dùng mutable tag và source code không readable
- uses: unknown/action@latest              # 'latest' = không thể predict behavior
```

### Checklist "Đừng Dùng"

- [ ] Action không có source code công khai (closed source)
- [ ] Action request `write` quyền mà không giải thích
- [ ] Action tải external scripts và execute mà không verify checksum
- [ ] Action gửi data (env vars, workspace) lên external endpoints không rõ
- [ ] Maintainer không respond issues trong > 6 tháng với nhiều open bugs
- [ ] Action có fewer than 50 stars nhưng được promote mạnh

---

## Chính Sách Enterprise

### Giới Hạn Actions Được Phép Dùng

GitHub Enterprise cho phép admin cấu hình policy (chính sách) cho phép actions:

```
Organization Settings → Actions → General → Actions permissions:
├── Allow all actions             (không recommended cho enterprise)
├── Allow local actions only      (chỉ actions trong cùng org)
├── Allow select actions          (whitelist cụ thể)
└── Disable actions               (không dùng Actions)
```

### Template Policy File (Tài Liệu Nội Bộ)

```markdown
# Chính Sách Sử Dụng GitHub Actions — [Tên Công Ty]

## Actions Được Phép Mặc Định

### Tier 1 — Luôn Được Phép
- Tất cả actions trong `actions/` organization (GitHub official)
- Tất cả actions trong `.github/` của cùng organization
- Cloud provider official: `aws-actions/*`, `google-github-actions/*`, `azure/*`

### Tier 2 — Được Phép Sau Khi Review
- Community actions với > 1000 stars, maintained, có license rõ ràng
- Phải pin theo SHA
- Phải qua security review của Platform Team

### Tier 3 — Cần Approval Đặc Biệt
- Actions từ unknown publisher
- Actions mới (< 3 tháng tuổi)
- Actions fork từ official actions

## Yêu Cầu Kỹ Thuật

1. **Pin theo SHA** cho tất cả Tier 2, Tier 3 actions
2. **Dependabot** phải được enable cho tất cả repos
3. **permissions block** phải ở mức tối thiểu trong mọi workflow
4. **Không dùng** `secrets: inherit` trong reusable workflows

## Quy Trình Thêm Action Mới

1. Submit request tại [link to internal form]
2. Platform Team review trong 5 ngày làm việc
3. Nếu approve → thêm vào allowlist
4. Nếu reject → Platform Team đề xuất alternative
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao pin actions theo SHA tốt hơn tag? Hạn chế là gì?**

A: SHA là immutable — một khi commit được tạo, SHA không bao giờ thay đổi. Tag có thể bị move (chủ ý hoặc do supply chain attack). Hạn chế: SHA khó đọc, không rõ phiên bản nào; giải pháp là thêm comment `# v4.2.2` cạnh SHA. Dependabot xử lý tốt cả SHA và tag khi có update mới.

**Q: Công ty bạn xử lý third-party actions như thế nào để đảm bảo bảo mật?**

A: Một approach tốt: (1) Phân loại actions theo tier — official, trusted community, unknown; (2) Yêu cầu pin theo SHA cho tier 2+ ; (3) Enable Dependabot để auto-update; (4) Có allowlist ở organization level; (5) Review source code trước khi approve action mới; (6) Theo dõi GitHub Security Advisories cho actions đang dùng.

**Q: Khi nào nên viết custom action thay vì dùng từ Marketplace?**

A: Viết custom khi: (1) Logic nghiệp vụ đặc thù của tổ chức; (2) Cần kết hợp nhiều bước thành một; (3) Marketplace action gần đúng nhưng cần customization quan trọng; (4) Action critical và cần full control; (5) Compliance yêu cầu code review nội bộ. Ưu tiên dùng Marketplace trước — không nên reinvent the wheel (tái phát minh bánh xe) cho các tác vụ phổ biến.

**Q: `uses: actions/checkout@v4` có an toàn không, hay nên pin SHA?**

A: Cho `actions/` organization (GitHub official), dùng `@v4` là acceptable — GitHub có quy trình bảo mật nghiêm ngặt và hiếm khi có supply chain attack. Tuy nhiên trong môi trường enterprise với compliance requirements, nên pin SHA cho mọi action. Đây là trade-off giữa security và maintainability — phụ thuộc vào risk tolerance của tổ chức.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
