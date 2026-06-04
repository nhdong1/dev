# 🔗 Supply Chain Security — Bảo Mật Chuỗi Cung Ứng

> Supply Chain Security — Bảo Mật Chuỗi Cung Ứng Phần Mềm là tập hợp các biện pháp bảo vệ pipeline CI/CD khỏi các dependencies (phụ thuộc) bên ngoài bị tấn công hoặc bị nhiễm độc, bao gồm việc pin actions bằng SHA, dùng Dependabot để theo dõi cập nhật, và dependency review cho mỗi PR.

---

## 📚 Mục Lục

1. [Supply Chain Attack Là Gì?](#supply-chain-attack-là-gì)
2. [Pin Actions Bằng Commit SHA](#pin-actions-bằng-commit-sha)
3. [Dependabot Cho GitHub Actions](#dependabot-cho-github-actions)
4. [Dependency Review Action](#dependency-review-action)
5. [Đánh Giá Actions Trước Khi Dùng](#đánh-giá-actions-trước-khi-dùng)
6. [SLSA Framework — Khung Bảo Mật Chuỗi Cung Ứng](#slsa-framework)
7. [Artifact Attestation — Chứng Nhận Tệp Đầu Ra](#artifact-attestation)

---

## Supply Chain Attack Là Gì?

### Định Nghĩa

**Supply Chain Attack** (Tấn công chuỗi cung ứng) là hình thức tấn công trong đó kẻ tấn công xâm nhập vào một dependency (phụ thuộc) của bạn — chứ không tấn công trực tiếp vào hệ thống của bạn — để inject malicious code (mã độc hại).

### Ví Dụ Thực Tế

#### Vụ tj-actions/changed-files (Tháng 3/2023)

```
1. Kẻ tấn công chiếm quyền kiểm soát GitHub account của maintainer
2. Thay đổi action để in tất cả secrets ra workflow logs
3. Hàng nghìn repository bị ảnh hưởng
4. Secrets bị lộ ra public logs

Bài học: Nếu đã pin bằng SHA → workflow không tự động nhận code mới
```

#### Các Vectơ Tấn Công Phổ Biến

```
┌─────────────────────────────────────────────────────────┐
│ 1. Compromised Action Account (Chiếm tài khoản action)  │
│    → Tag v1, v2 bị đổi sang code mới                   │
│                                                          │
│ 2. Typosquatting (Đánh lừa tên gần giống)               │
│    → actions/checkoutt (thêm 't' giả mạo)              │
│                                                          │
│ 3. Dependency Confusion (Nhầm lẫn phụ thuộc)            │
│    → Package nội bộ bị thay thế bởi package public      │
│                                                          │
│ 4. Malicious PR Merged (PR độc hại được merge)          │
│    → Code xấu vào action được dùng bởi nhiều repos     │
└─────────────────────────────────────────────────────────┘
```

---

## Pin Actions Bằng Commit SHA

### Tại Sao SHA An Toàn Hơn Version Tag?

```
Version Tag (v4):
  - Có thể bị xóa và tái tạo trỏ sang commit khác
  - Kẻ tấn công chiếm tài khoản → thay đổi v4 sang code độc hại
  - Bạn không thay đổi gì trong workflow nhưng vẫn bị ảnh hưởng

Commit SHA (e.g., 11bd71901bbe5b1630ceea73d27597364c9af683):
  - Bất biến (immutable) — không thể thay đổi
  - Nếu commit đó tồn tại → nội dung giữ nguyên mãi mãi
  - Kẻ tấn công phải push commit mới → SHA sẽ khác
```

### Cú Pháp Pin Bằng SHA

```yaml
# ❌ KHÔNG AN TOÀN — tag có thể bị thay đổi
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
- uses: actions/upload-artifact@v4

# ✅ AN TOÀN — SHA không thể thay đổi
# Thêm comment để biết đây là version nào
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
- uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4.6.2
```

### Cách Tìm SHA Của Một Action

#### Phương Pháp 1: GitHub UI

```
1. Vào trang action: github.com/actions/checkout
2. Chọn tag v4
3. Click "X commits to main" hoặc xem Releases
4. Copy full SHA của commit tương ứng
```

#### Phương Pháp 2: GitHub API

```bash
# Lấy SHA của tag v4 từ actions/checkout
curl -s https://api.github.com/repos/actions/checkout/git/ref/tags/v4 \
  | jq -r '.object.sha'

# Nếu tag là annotated tag, cần thêm bước
COMMIT_SHA=$(curl -s https://api.github.com/repos/actions/checkout/git/tags/TAG_SHA \
  | jq -r '.object.sha')
```

#### Phương Pháp 3: Công Cụ Tự Động

```bash
# Dùng pin-github-action tool
pip install pin-github-action
pin-github-action .github/workflows/ci.yml

# Dùng safer-actions-versions
npx safer-actions-versions
```

### Workflow Hoàn Chỉnh Với SHA Pins

```yaml
name: CI Pipeline An Toàn

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Setup Node.js
        uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
        with:
          node-version: '22'
          cache: 'npm'

      - name: Cài đặt dependencies
        run: npm ci

      - name: Chạy tests
        run: npm test

      - name: Upload coverage
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4.6.2
        with:
          name: coverage-report
          path: coverage/
```

---

## Dependabot Cho GitHub Actions

### Bật Dependabot Tự Động Cập Nhật

Tạo file `.github/dependabot.yml`:

```yaml
version: 2

updates:
  # Cập nhật GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"          # Kiểm tra hàng tuần
      day: "monday"               # Vào thứ Hai
      time: "09:00"               # Lúc 9:00 sáng
      timezone: "Asia/Ho_Chi_Minh"
    labels:
      - "dependencies"
      - "github-actions"
    reviewers:
      - "team-devops"             # Yêu cầu review từ team
    assignees:
      - "devops-lead"
    commit-message:
      prefix: "chore(deps)"       # Tiền tố commit message
    open-pull-requests-limit: 10  # Giới hạn số PRs mở đồng thời

  # Cập nhật npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "javascript"
    ignore:
      - dependency-name: "lodash"  # Bỏ qua package cụ thể
        versions: ["4.x"]

  # Cập nhật Docker base images
  - package-ecosystem: "docker"
    directory: "/docker"
    schedule:
      interval: "weekly"
```

### Tự Động Merge Dependabot PRs An Toàn

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot Auto-merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot:
    runs-on: ubuntu-latest
    # Chỉ chạy cho Dependabot PRs
    if: github.actor == 'dependabot[bot]'

    steps:
      - name: Lấy metadata của Dependabot PR
        id: metadata
        uses: dependabot/fetch-metadata@dbb049abf0d677abbd7f7eee0375145b417fdd34 # v2.3.0
        with:
          github-token: "${{ secrets.GITHUB_TOKEN }}"

      - name: Auto-merge patch updates (cập nhật vá lỗi nhỏ)
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Major updates cần manual review
      - name: Comment yêu cầu review cho major update
        if: steps.metadata.outputs.update-type == 'version-update:semver-major'
        run: |
          gh pr comment "$PR_URL" --body "⚠️ **Major update** — Cần review thủ công trước khi merge."
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Dependency Review Action

### Kiểm Tra Phụ Thuộc Mới Trong PR

```yaml
# .github/workflows/dependency-review.yml
name: Dependency Review

on:
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  pull-requests: write  # Để comment kết quả vào PR

jobs:
  dependency-review:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Dependency Review
        uses: actions/dependency-review-action@da24c61d1e95cfc0e08bb4b4fe1a5b5a69f5fe49 # v4.5.0
        with:
          # Thất bại nếu có vulnerability mức cao hoặc nghiêm trọng
          fail-on-severity: high

          # Cho phép các license này
          allow-licenses: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC

          # Từ chối các license này
          deny-licenses: GPL-2.0, GPL-3.0, LGPL-2.0, LGPL-2.1

          # Comment kết quả vào PR
          comment-summary-in-pr: always

          # Thất bại nếu có GHSA advisories bị ghim
          # deny-ghsas: GHSA-xxxx-xxxx-xxxx
```

### Kết Hợp Với License Compliance (Tuân Thủ Giấy Phép)

```yaml
- name: Dependency Review với License Check
  uses: actions/dependency-review-action@da24c61d1e95cfc0e08bb4b4fe1a5b5a69f5fe49 # v4.5.0
  with:
    fail-on-severity: moderate
    allow-licenses: >-
      MIT,
      Apache-2.0,
      BSD-2-Clause,
      BSD-3-Clause,
      ISC,
      0BSD,
      CC0-1.0,
      Unlicense
    # Tùy chọn: chỉ kiểm tra ecosystem cụ thể
    # allow-ghsas: GHSA-xxxx
    # Tùy chọn: bỏ qua dependency cụ thể
    # config-file: .github/dependency-review-config.yml
```

---

## Đánh Giá Actions Trước Khi Dùng

### Tiêu Chí Đánh Giá Action

```
✅ Nguồn gốc đáng tin cậy
  □ Actions từ github.com/actions (chính thức)
  □ Actions từ tổ chức lớn (aws-actions, google-github-actions, azure)
  □ Actions có nhiều stars (>1000), được maintain tích cực
  □ Source code có thể kiểm tra công khai

✅ Bảo mật
  □ Permissions tối thiểu (không yêu cầu write-all)
  □ Không đọc/ghi secrets không cần thiết
  □ Không gọi external endpoints không rõ ràng
  □ Có security policy rõ ràng

✅ Chất lượng code
  □ Code được review kỹ, không có code rõ ràng độc hại
  □ Dependencies được pin version
  □ Được test đầy đủ (CI xanh)
  □ Không dùng eval() hoặc exec() với input không kiểm soát

✅ Lịch sử bảo trì
  □ Commit gần đây (trong 6 tháng)
  □ Issues được phản hồi
  □ Không có open security vulnerabilities
```

### Kiểm Tra Nhanh Một Action

```bash
# 1. Kiểm tra source code trực tiếp
# Vào github.com/[org]/[action] → xem action.yml hoặc src/

# 2. Kiểm tra permissions action yêu cầu
grep -A 20 "permissions:" action.yml

# 3. Kiểm tra network calls trong code
grep -r "fetch\|axios\|request\|http\|curl" src/ --include="*.js" --include="*.ts"

# 4. Xem các releases và release notes
# github.com/[org]/[action]/releases

# 5. Kiểm tra GHSA advisories
# github.com/advisories?query=[action-name]
```

### Khi Nào Nên Fork Action

```yaml
# Cân nhắc fork khi:
# 1. Action không được maintain tích cực
# 2. Action từ source không rõ ràng
# 3. Cần thay đổi nhỏ nhưng không muốn phụ thuộc upstream

# Sau khi fork:
- uses: my-org/forked-action@SHA_OF_VERIFIED_COMMIT
  # Chỉ cần đảm bảo bạn review code tại SHA đó
```

---

## SLSA Framework

### SLSA — Supply-chain Levels for Software Artifacts — Các Cấp Độ Bảo Mật Chuỗi Cung Ứng Phần Mềm

**SLSA** (phát âm "salsa") là framework của Google để đảm bảo tính toàn vẹn của chuỗi cung ứng phần mềm.

```
SLSA Level 1: Documentation (Tài Liệu)
  → Build process được tài liệu hóa, có provenance (nguồn gốc)
  → Cơ bản nhất, dễ đạt

SLSA Level 2: Hosted (Trên Nền Tảng Được Kiểm Soát)
  → Build chạy trên hosted platform (GitHub Actions, Google Cloud Build)
  → Provenance được ký bởi platform

SLSA Level 3: Hardened Build (Build Được Tăng Cường)
  → Build không thể bị ảnh hưởng bởi người dùng
  → Provenance được ký bởi build platform không thể giả mạo

SLSA Level 4 (Future): Two-party Review
  → Mọi thay đổi phải được review và approve bởi 2 người trở lên
```

### Tạo SLSA Provenance Với GitHub Actions

```yaml
name: Build và Generate SLSA Provenance

on:
  push:
    tags: ['v*']

permissions:
  contents: write
  id-token: write      # Cần cho SLSA signing
  attestations: write  # Cần cho artifact attestation

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digests: ${{ steps.hash.outputs.digests }}

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Build artifacts
        run: |
          make build
          sha256sum dist/* > checksums.txt

      - name: Upload artifacts
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4.6.2
        with:
          name: dist
          path: dist/

      - name: Hash artifacts (tạo hash để verify)
        id: hash
        run: |
          DIGESTS=$(sha256sum dist/* | base64 -w 0)
          echo "digests=$DIGESTS" >> $GITHUB_OUTPUT

  provenance:
    needs: [build]
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.digests }}"
      upload-assets: true
    permissions:
      actions: read
      id-token: write
      contents: write
```

---

## Artifact Attestation — Chứng Nhận Tệp Đầu Ra

### Tạo và Verify Artifact Attestation

```yaml
name: Build Docker Image với Attestation

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write  # Mới từ GitHub 2024

jobs:
  build-and-attest:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Build Docker image
        id: build
        uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # v6.18.0
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Tạo attestation cho image
        uses: actions/attest-build-provenance@c074443f1a9c96cc1cb551c0cf88f7ce31c44d5b # v2.2.3
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
```

### Verify Attestation

```bash
# Verify attestation của một image
gh attestation verify \
  oci://ghcr.io/my-org/my-repo:latest \
  --owner my-org

# Verify một file cụ thể
gh attestation verify my-binary \
  --owner my-org
```

---

## 📋 Checklist Supply Chain Security

### Mức Cơ Bản

- [ ] Pin tất cả third-party actions bằng commit SHA (không dùng version tags)
- [ ] Thêm comment SHA → version để dễ đọc (ví dụ: `# v4.2.2`)
- [ ] Bật Dependabot cho `github-actions` ecosystem
- [ ] Review Dependabot PRs trước khi merge

### Mức Trung Cấp

- [ ] Cài đặt Dependency Review Action trong PR workflows
- [ ] Cấu hình license allowlist và denylist
- [ ] Thiết lập fail-on-severity cho vulnerability scanning
- [ ] Kiểm tra source code của third-party actions trước khi dùng lần đầu

### Mức Nâng Cao

- [ ] Generate SLSA provenance cho production builds
- [ ] Implement artifact attestation cho Docker images
- [ ] Thiết lập policy kiểm tra attestation trước khi deploy
- [ ] Audit lại tất cả actions đang dùng hàng quý

---

## 🔗 Xem Thêm

- [1-permissions.md](./1-permissions.md) — Quyền tối thiểu
- [4-code-scanning.md](./4-code-scanning.md) — CodeQL và SAST scanning
- [6-security-hardening.md](./6-security-hardening.md) — Hardening checklist
- [05-reusable/5-marketplace-guide.md](../05-reusable/5-marketplace-guide.md) — Đánh giá Marketplace actions

---

**Cập Nhật Lần Cuối:** 2026-05-12
