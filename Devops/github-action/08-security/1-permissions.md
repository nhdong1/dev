# 🔑 Permissions Block & GITHUB_TOKEN Scopes

> Kiểm soát quyền truy cập tối thiểu (least-privilege — Quyền Tối Thiểu) cho mọi workflow bằng `permissions` block và hiểu rõ phạm vi của GITHUB_TOKEN (Token Tự Động GitHub).

---

## 📚 Mục Lục

1. [GITHUB_TOKEN Là Gì?](#github_token-là-gì)
2. [Permissions Block Cú Pháp](#permissions-block-cú-pháp)
3. [Tất Cả Scopes Có Sẵn](#tất-cả-scopes-có-sẵn)
4. [Quyền Mặc Định Theo Loại Event](#quyền-mặc-định-theo-loại-event)
5. [Cấu Hình Repository Default Permissions](#cấu-hình-repository-default-permissions)
6. [Pattern Thực Tế Theo Use Case](#pattern-thực-tế-theo-use-case)
7. [Lỗi Phổ Biến & Cách Tránh](#lỗi-phổ-biến--cách-tránh)

---

## GITHUB_TOKEN Là Gì?

`GITHUB_TOKEN` là **JWT token** (JSON Web Token — Mã Thông Báo Web JSON) được GitHub tự động tạo ra cho mỗi workflow run. Đây là cơ chế xác thực mặc định để workflow tương tác với GitHub API.

### Đặc Điểm Kỹ Thuật

| Thuộc Tính | Giá Trị |
|---|---|
| **Loại** | JWT (JSON Web Token) |
| **Thời hạn** | Hết hạn khi workflow kết thúc hoặc tối đa 24 giờ |
| **Phạm vi** | Chỉ repository hiện tại |
| **Tái tạo** | Mới mỗi workflow run |
| **Vị trí** | `${{ secrets.GITHUB_TOKEN }}` hoặc `${{ github.token }}` |

### Cách GITHUB_TOKEN Hoạt Động

```
┌─────────────────────────────────────────────────────────────┐
│  1. Workflow được trigger                                    │
│  2. GitHub Actions tạo GITHUB_TOKEN mới                     │
│  3. Token được inject vào workflow run environment          │
│  4. Workflow sử dụng token để gọi GitHub API                │
│  5. Workflow kết thúc → token bị thu hồi                    │
└─────────────────────────────────────────────────────────────┘
```

### Truy Cập GITHUB_TOKEN Trong Workflow

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      # Cách 1: Qua secrets context
      - name: Gọi GitHub API
        run: |
          curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${{ github.repository }}/issues

      # Cách 2: Qua github context (ngắn hơn)
      - name: Gọi GitHub API (cách ngắn)
        run: |
          curl -H "Authorization: Bearer ${{ github.token }}" \
            https://api.github.com/repos/${{ github.repository }}/issues

      # Cách 3: Truyền vào action qua input
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Permissions Block Cú Pháp

### Cấp Workflow (Áp Dụng Cho Tất Cả Jobs)

```yaml
name: Workflow Với Permissions Rõ Ràng

on: push

# Khai báo ở cấp workflow — áp dụng cho tất cả jobs
permissions:
  contents: read       # Đọc code repository
  pull-requests: write # Ghi comment vào PR

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

### Cấp Job (Ghi Đè Permissions Của Workflow)

```yaml
name: Workflow Với Job-Level Permissions

on: push

permissions: read-all  # Mặc định: chỉ đọc cho toàn workflow

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read    # Chỉ cần đọc code
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

  deploy:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: read
      deployments: write  # Job này cần thêm quyền deployments
    steps:
      - name: Tạo deployment
        run: echo "Deploying..."
```

### Giá Trị Có Thể Dùng

```yaml
permissions:
  scope-name: read    # Chỉ đọc
  scope-name: write   # Đọc và ghi
  scope-name: none    # Không có quyền gì

# Shorthand
permissions: read-all   # Tất cả scopes đều read
permissions: write-all  # Tất cả scopes đều write (NGUY HIỂM)
permissions: {}         # Không có quyền gì (an toàn nhất)
```

---

## Tất Cả Scopes Có Sẵn

| Scope | Quyền Mặc Định | Mô Tả |
|---|---|---|
| `actions` | read/write | Quản lý Actions — workflows, artifacts, runs |
| `attestations` | none | Tạo artifact attestations (chứng nhận artifact) |
| `checks` | read/write | Tạo và cập nhật check runs, check suites |
| `contents` | read/write | Đọc/ghi code, releases, tags, commits |
| `deployments` | read/write | Tạo và cập nhật deployments |
| `discussions` | read/write | Tạo và quản lý GitHub Discussions |
| `id-token` | none | Yêu cầu OIDC JWT token từ provider |
| `issues` | read/write | Tạo, đọc, cập nhật issues và comments |
| `packages` | read/write | Publish/download từ GitHub Packages |
| `pages` | read/write | Tạo và quản lý GitHub Pages |
| `pull-requests` | read/write | Tạo, đọc, cập nhật PRs và reviews |
| `repository-projects` | read/write | Quản lý GitHub Projects (classic) |
| `security-events` | read/write | Đọc/ghi Security Advisories, code scanning |
| `statuses` | read/write | Tạo commit statuses (trạng thái commit) |

### Scope Quan Trọng Nhất: `id-token`

```yaml
# Cần thiết khi dùng OIDC để xác thực với cloud providers
permissions:
  id-token: write   # Phải có write để request OIDC token
  contents: read
```

---

## Quyền Mặc Định Theo Loại Event

GitHub tự động điều chỉnh permissions mặc định dựa trên loại event trigger:

### Push & Schedule Events

```
contents: write
pull-requests: read
packages: write
deployments: write
```

### Pull Request Từ Fork (Fork PR)

```
# Giảm xuống chỉ read-only vì code đến từ bên ngoài
contents: read
pull-requests: read
```

### Pull Request Target

```
# CẢNH BÁO: pull_request_target chạy với write access!
# Mặc dù code đến từ fork, workflow chạy trong ngữ cảnh base repo
contents: write  # NGUY HIỂM với code từ fork!
```

### Workflow Dispatch & Call

```
# Kế thừa từ cấu hình workflow
# Luôn khai báo tường minh
```

---

## Cấu Hình Repository Default Permissions

### Qua GitHub UI

```
Repository Settings
  → Actions
    → General
      → Workflow permissions
        ○ Read and write permissions (Đọc và ghi — mặc định cũ)
        ● Read repository contents and packages permissions (Chỉ đọc — khuyến nghị)
```

### Qua GitHub CLI (gh)

```bash
# Xem permissions hiện tại
gh api repos/OWNER/REPO/actions/permissions

# Đặt default permissions thành read-only
gh api --method PUT repos/OWNER/REPO/actions/permissions \
  --field default_workflow_permissions=read \
  --field can_approve_pull_request_reviews=false
```

---

## Pattern Thực Tế Theo Use Case

### Pattern 1: CI Pipeline Chỉ Đọc

```yaml
name: CI — Lint và Test

on:
  pull_request:
    branches: [main, develop]

permissions:
  contents: read          # Checkout code
  checks: write           # Ghi kết quả test vào Checks UI
  pull-requests: write    # Comment kết quả coverage vào PR

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Chạy tests
        run: npm test

      - name: Comment coverage vào PR
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Tests passed!'
            })
```

### Pattern 2: Release Workflow

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write   # Tạo GitHub Release, upload assets
  packages: write   # Push Docker image lên GitHub Packages

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Tạo GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
```

### Pattern 3: Dependency Review

```yaml
name: Dependency Review

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write  # Để post review comment

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/dependency-review-action@da24c61d1e95cfc0e08bb4b4fe1a5b5a69f5fe49 # v4.5.0
```

### Pattern 4: Code Scanning (OIDC + Security Events)

```yaml
name: Code Scanning

on:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write  # Upload kết quả SARIF lên GitHub Security
  id-token: write         # OIDC (nếu cần xác thực cloud)

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Initialize CodeQL
        uses: github/codeql-action/init@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        with:
          languages: javascript

      - name: Analyze
        uses: github/codeql-action/analyze@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
```

### Pattern 5: Deployment Với Minimal Permissions

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

# Khóa tất cả permissions ở cấp workflow
permissions: {}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write   # Chỉ cần OIDC để lấy cloud credentials
      contents: read    # Chỉ đọc code
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1

      - name: Deploy
        run: aws ecs update-service --cluster prod --service app --force-new-deployment
```

---

## Lỗi Phổ Biến & Cách Tránh

### Lỗi 1: Quên Khai Báo `id-token: write` Khi Dùng OIDC

```yaml
# ❌ LỖI — thiếu id-token permission
permissions:
  contents: read

# ✅ ĐÚNG
permissions:
  contents: read
  id-token: write  # Bắt buộc khi dùng OIDC
```

### Lỗi 2: Dùng `write-all` Không Cần Thiết

```yaml
# ❌ NGUY HIỂM — cấp tất cả quyền ghi
permissions: write-all

# ✅ ĐÚNG — chỉ cấp những gì cần
permissions:
  contents: read
  pull-requests: write
```

### Lỗi 3: Không Khai Báo Permissions Tường Minh

```yaml
# ❌ RỦI RO — phụ thuộc vào repo default settings
name: My Workflow
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building..."

# ✅ ĐÚNG — luôn khai báo rõ ràng
name: My Workflow
on: push
permissions:
  contents: read    # Tường minh — không phụ thuộc defaults
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building..."
```

### Lỗi 4: Dùng `pull_request_target` Với Code Từ Fork

```yaml
# ❌ NGUY HIỂM — checkout code từ fork với write permissions
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      # NGUY HIỂM: checkout code từ fork (có thể chứa malicious code)
      # Nhưng workflow chạy với write access của base repo
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

# ✅ AN TOÀN — dùng pull_request thay thế
on:
  pull_request:
    types: [opened, synchronize]
# Hoặc nếu bắt buộc phải dùng pull_request_target:
# Chỉ checkout code từ base repo, không checkout fork code
```

---

## 🔍 Kiểm Tra Permissions Hiện Tại

```bash
# Xem permissions của GITHUB_TOKEN trong một workflow run
# Thêm step này vào workflow để debug
- name: Debug permissions
  run: |
    curl -s -H "Authorization: Bearer ${{ github.token }}" \
      https://api.github.com/repos/${{ github.repository }} \
      | jq '.permissions'
```

---

## 📊 So Sánh GITHUB_TOKEN vs PAT vs OIDC

| Tiêu Chí | GITHUB_TOKEN | PAT (Personal Access Token) | OIDC |
|---|---|---|---|
| **Phạm vi** | Chỉ 1 repo | Nhiều repos | Cloud resources |
| **Thời hạn** | Hết khi workflow xong | Tháng/năm | Phút |
| **Rủi ro lộ** | Thấp (tự động xóa) | Cao | Rất thấp |
| **Cấu hình** | Tự động | Phải tạo thủ công | Cần setup OIDC trust |
| **Dùng cho** | GitHub API | Cross-repo, GitHub API | Cloud providers |

---

## 🔗 Xem Thêm

- [2-oidc-cloud-auth.md](./2-oidc-cloud-auth.md) — OIDC với cloud providers
- [6-security-hardening.md](./6-security-hardening.md) — Checklist bảo mật toàn diện
- [GitHub Docs: Permissions for the GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)

---

**Cập Nhật Lần Cuối:** 2026-05-12
