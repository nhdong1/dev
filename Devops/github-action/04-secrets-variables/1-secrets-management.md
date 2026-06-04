# 🔑 Secrets Management — Quản Lý Bí Mật Trong GitHub Actions

> Hướng dẫn đầy đủ về GitHub Secrets — cách tạo, phạm vi, truy cập, best practices và các lỗi phổ biến khi quản lý thông tin nhạy cảm.

---

## 📚 Mục Lục

1. [GitHub Secrets Là Gì?](#github-secrets-là-gì)
2. [Ba Cấp Độ Secrets](#ba-cấp-độ-secrets)
3. [Repository Secrets](#repository-secrets)
4. [Organization Secrets](#organization-secrets)
5. [Environment Secrets](#environment-secrets)
6. [Truy Cập Secrets Trong Workflow](#truy-cập-secrets-trong-workflow)
7. [GITHUB_TOKEN — Token Đặc Biệt](#github_token--token-đặc-biệt)
8. [Bảo Mật Secrets](#bảo-mật-secrets)
9. [Secret Rotation — Xoay Vòng Bí Mật](#secret-rotation--xoay-vòng-bí-mật)
10. [Lỗi Phổ Biến và Cách Tránh](#lỗi-phổ-biến-và-cách-tránh)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## GitHub Secrets Là Gì?

GitHub Secrets là cơ chế lưu trữ thông tin nhạy cảm được mã hóa (encrypted) trong GitHub. Khi workflow chạy, secrets được giải mã trong bộ nhớ runner và **không bao giờ ghi vào disk hay hiển thị trong logs**.

### Cách GitHub Bảo Vệ Secrets

```
┌─────────────────────────────────────────────────────────────┐
│                   GitHub Secrets Security                   │
│                                                             │
│  You Enter Secret                                           │
│       │                                                     │
│       ▼                                                     │
│  [Encrypted with libsodium — NaCl box encryption]          │
│       │                                                     │
│       ▼                                                     │
│  [Stored in GitHub's encrypted secrets store]              │
│       │                                                     │
│       ▼ (at workflow run time)                              │
│  [Decrypted in-memory on the runner]                        │
│       │                                                     │
│       ▼                                                     │
│  Available as ${{ secrets.MY_SECRET }}                      │
│       │                                                     │
│       ▼                                                     │
│  [Value masked — *** — in all log output]                   │
└─────────────────────────────────────────────────────────────┘
```

### Giới Hạn Kỹ Thuật

| Thuộc Tính | Giá Trị |
|---|---|
| Kích thước tối đa mỗi secret | 48 KB |
| Số secrets tối đa per repository | 100 |
| Số secrets tối đa per organization | 1,000 |
| Số secrets tối đa per environment | 100 |
| Tên secret | Chỉ chữ cái, số, dấu gạch dưới; không bắt đầu bằng `GITHUB_` |

---

## Ba Cấp Độ Secrets

```
Organization Secrets (Cấp Tổ Chức)
    ├── Áp dụng cho nhiều repositories trong cùng organization
    ├── Quản lý tập trung
    └── Có thể giới hạn repos nào được dùng

Repository Secrets (Cấp Repository)
    ├── Chỉ dùng trong một repository cụ thể
    ├── Overrides — ghi đè organization secrets cùng tên
    └── Dễ quản lý nhất

Environment Secrets (Cấp Môi Trường)
    ├── Gắn với một environment cụ thể (staging, production)
    ├── Chỉ accessible khi job target environment đó
    ├── Overrides repository và organization secrets
    └── Kết hợp với protection rules cho extra security
```

### Thứ Tự Ưu Tiên (Overriding)

```
Environment Secrets  >  Repository Secrets  >  Organization Secrets
(Cao nhất)                                      (Thấp nhất)
```

---

## Repository Secrets

### Tạo Repository Secret Qua UI

```
GitHub Repository → Settings → Secrets and variables → Actions → New repository secret
```

### Tạo Qua GitHub CLI

```bash
# Tạo secret từ giá trị trực tiếp
gh secret set MY_SECRET --body "my-secret-value"

# Tạo secret từ file (cho private keys, certificates)
gh secret set PRIVATE_KEY < private.pem

# Tạo secret với repo cụ thể
gh secret set MY_SECRET --repo owner/repo --body "value"

# Liệt kê secrets (chỉ hiện tên, không hiện giá trị)
gh secret list

# Xóa secret
gh secret delete MY_SECRET
```

### Tạo Qua GitHub API

```bash
# 1. Lấy public key của repo để mã hóa secret
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/secrets/public-key

# Response:
# {
#   "key_id": "012345678912345678",
#   "key": "2Sg8iYjAxxmI2LvUXpJjkYrMxURPc8r+dB7TJyvv1234"
# }

# 2. Mã hóa giá trị secret với libsodium
# 3. Gửi lên API
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  https://api.github.com/repos/OWNER/REPO/actions/secrets/MY_SECRET \
  -d '{"encrypted_value":"c2VjcmV0","key_id":"012345678912345678"}'
```

---

## Organization Secrets

### Tạo Organization Secret

```
GitHub Organization → Settings → Secrets and variables → Actions → New organization secret
```

### Cấu Hình Repository Access

Khi tạo organization secret, chọn:
- **All repositories** — tất cả repos trong org có thể dùng
- **Private repositories** — chỉ repos private
- **Selected repositories** — chọn từng repo cụ thể

### Dùng Organization Secret Trong Workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Use shared credentials
        env:
          # Organization secret — tự động có sẵn nếu repo được phép
          SHARED_API_KEY: ${{ secrets.SHARED_API_KEY }}
        run: |
          echo "Using shared API key..."
```

### CLI: Tạo Organization Secret

```bash
# Tạo org secret với visibility cho all repos
gh secret set SHARED_TOKEN \
  --org my-organization \
  --visibility all \
  --body "shared-token-value"

# Tạo với selected repos
gh secret set SHARED_TOKEN \
  --org my-organization \
  --visibility selected \
  --repos "repo1,repo2,repo3" \
  --body "shared-token-value"
```

---

## Environment Secrets

Environment Secrets cung cấp layer bảo mật bổ sung — secret chỉ có thể dùng khi job chạy trong context của một environment cụ thể.

### Tạo Environment

```
Repository → Settings → Environments → New environment
```

### Cấu Hình Environment Với Protection Rules

```
Environment "production":
  ├── Required reviewers: [lead-engineer, security-team]
  ├── Wait timer: 30 minutes
  ├── Deployment branches: main only
  └── Secrets:
      ├── PROD_DATABASE_URL
      ├── PROD_API_KEY
      └── PROD_DEPLOY_TOKEN
```

### Workflow Dùng Environment Secrets

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Chỉ định environment — kích hoạt protection rules
    steps:
      - name: Deploy với production secrets
        env:
          DB_URL: ${{ secrets.PROD_DATABASE_URL }}    # Environment secret
          API_KEY: ${{ secrets.PROD_API_KEY }}        # Environment secret
        run: |
          echo "Deploying with production credentials..."
          ./scripts/deploy.sh
```

### Multi-Environment Pipeline

```yaml
name: Multi-Stage Deployment

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        env:
          DB_URL: ${{ secrets.DATABASE_URL }}  # Staging DATABASE_URL
          API_KEY: ${{ secrets.API_KEY }}      # Staging API_KEY
        run: ./deploy.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    environment: production                   # Khác environment → khác secrets!
    steps:
      - name: Deploy to Production
        env:
          DB_URL: ${{ secrets.DATABASE_URL }}  # Production DATABASE_URL (cùng tên, khác giá trị)
          API_KEY: ${{ secrets.API_KEY }}      # Production API_KEY
        run: ./deploy.sh production
```

---

## Truy Cập Secrets Trong Workflow

### Cú Pháp Cơ Bản

```yaml
steps:
  # Cách 1: Truyền qua env variable
  - name: Use secret via env
    env:
      MY_TOKEN: ${{ secrets.MY_TOKEN }}
    run: |
      echo "Token is available as $MY_TOKEN"

  # Cách 2: Truyền trực tiếp vào input của action
  - name: Use secret as input
    uses: some-action@v1
    with:
      token: ${{ secrets.MY_TOKEN }}

  # Cách 3: Inline trong run (KHÔNG KHUYẾN NGHỊ — dễ bị lộ)
  - name: NOT RECOMMENDED
    run: curl -H "Authorization: Bearer ${{ secrets.MY_TOKEN }}" https://api.example.com
```

### Truyền Secrets Sang Reusable Workflows

```yaml
# Caller workflow (workflow gọi)
jobs:
  deploy:
    uses: ./.github/workflows/deploy-reusable.yml
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
      # Hoặc truyền tất cả secrets
    secrets: inherit  # Kế thừa tất cả secrets từ caller

---
# Reusable workflow (workflow được gọi)
on:
  workflow_call:
    secrets:
      deploy-token:
        required: true
        description: "Token for deployment"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Using ${{ secrets.deploy-token }}"
```

### Multiline Secrets (Secrets Nhiều Dòng)

```yaml
# Private keys, certificates thường có nhiều dòng
# Lưu toàn bộ trong một secret

steps:
  - name: Setup SSH key
    run: |
      mkdir -p ~/.ssh
      echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
      chmod 600 ~/.ssh/id_rsa
```

---

## GITHUB_TOKEN — Token Đặc Biệt

`GITHUB_TOKEN` là secret đặc biệt được GitHub tự động tạo cho mỗi workflow run. Không cần tạo thủ công.

### Phạm Vi Quyền Mặc Định

```yaml
# Quyền mặc định của GITHUB_TOKEN (read-only cho hầu hết)
permissions:
  actions: read
  checks: read
  contents: read
  deployments: read
  id-token: none    # Cần bật thủ công cho OIDC
  issues: read
  packages: read
  pull-requests: read
  security-events: read
  statuses: read
```

### Cấu Hình Permissions

```yaml
name: CI with Custom Permissions

permissions:           # Cấp workflow level
  contents: read       # Chỉ đọc code
  pull-requests: write # Cho phép comment PR
  issues: write        # Cho phép tạo/update issues

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:       # Cấp job level — ghi đè workflow level
      contents: write  # Job này cần write để push artifacts
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}  # Dùng GITHUB_TOKEN
```

### Giá Trị Permissions Hợp Lệ

```
read    → Quyền đọc
write   → Quyền đọc + ghi
none    → Không có quyền gì
```

### Ví Dụ Thực Tế: Auto-comment PR

```yaml
name: PR Auto Comment

on:
  pull_request:
    types: [opened]

permissions:
  pull-requests: write  # Cần để comment

jobs:
  comment:
    runs-on: ubuntu-latest
    steps:
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '👋 Cảm ơn đã tạo PR! Team sẽ review sớm.'
            })
```

---

## Bảo Mật Secrets

### Best Practices (Thực Hành Tốt Nhất)

**1. Không bao giờ hardcode secrets trong code:**
```yaml
# ❌ SAI — Tuyệt đối không làm
- run: aws configure set aws_access_key_id AKIA1234567890ABCDEF

# ✅ ĐÚNG
- run: aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
```

**2. Không log secrets:**
```yaml
# ❌ SAI — Secret sẽ bị mask nhưng không nên làm
- run: echo "My secret is ${{ secrets.MY_SECRET }}"

# ✅ ĐÚNG — Chỉ dùng secret trong lệnh cần thiết
- run: ./script-that-uses-secret.sh
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
```

**3. Đặt tên rõ ràng cho secrets:**
```
# ❌ SAI
TOKEN
KEY
PASSWORD

# ✅ ĐÚNG
AWS_ACCESS_KEY_ID
DOCKER_HUB_TOKEN
PROD_DATABASE_PASSWORD
```

**4. Kiểm tra secret scanning (Quét Bí Mật):**
```yaml
# Bật secret scanning trong repository settings
# GitHub tự động phát hiện nếu secrets bị commit vào code
```

**5. Dùng environment secrets cho production:**
```yaml
# Production secrets chỉ accessible khi environment = "production"
# Với required reviewers → cần approval trước khi chạy
environment: production
```

### Phát Hiện Secret Bị Lộ

```bash
# GitHub tự động scan khi secret pattern bị phát hiện trong commits
# Bạn sẽ nhận email alert

# Tools để scan codebase trước khi commit:
# - git-secrets
# - truffleHog
# - detect-secrets (Yelp)
# - GitGuardian

# Pre-commit hook
git secrets --register-aws
git secrets --scan
```

---

## Secret Rotation — Xoay Vòng Bí Mật

Secret Rotation (Xoay Vòng Bí Mật) là quá trình định kỳ thay đổi giá trị secrets để giảm thiểu rủi ro nếu bị lộ.

### Rotation Thủ Công

```bash
#!/bin/bash
# Script xoay vòng secrets

# 1. Tạo credentials mới
NEW_API_KEY=$(generate-new-key)

# 2. Cập nhật GitHub Secret
gh secret set API_KEY --body "$NEW_API_KEY"

# 3. Xác nhận pipeline vẫn hoạt động
gh workflow run test-pipeline.yml

# 4. Thu hồi (revoke) credentials cũ
revoke-old-key "$OLD_API_KEY"
```

### Tự Động Hóa Rotation

```yaml
name: Rotate AWS Credentials

on:
  schedule:
    - cron: '0 0 1 * *'  # Chạy vào ngày 1 mỗi tháng

jobs:
  rotate:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Cần cho OIDC
    steps:
      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/SecretRotator
          aws-region: us-east-1

      - name: Rotate and update secrets
        run: |
          # Tạo access key mới
          NEW_KEY=$(aws iam create-access-key --user-name github-ci \
            --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text)
          
          NEW_KEY_ID=$(echo "$NEW_KEY" | cut -f1)
          NEW_KEY_SECRET=$(echo "$NEW_KEY" | cut -f2)
          
          # Cập nhật GitHub Secrets
          gh secret set AWS_ACCESS_KEY_ID --body "$NEW_KEY_ID"
          gh secret set AWS_SECRET_ACCESS_KEY --body "$NEW_KEY_SECRET"
          
          echo "Rotation completed. New key ID: $NEW_KEY_ID"
        env:
          GH_TOKEN: ${{ secrets.ROTATION_PAT }}  # PAT với secret write permission
```

---

## Lỗi Phổ Biến và Cách Tránh

### Lỗi 1: Secret bị expose qua third-party actions

```yaml
# ❌ RỦI RO — Action không tin cậy có thể đọc env variables
- uses: untrusted-action@v1
  env:
    ALL_MY_SECRETS: ${{ toJSON(secrets) }}  # Tuyệt đối không làm

# ✅ AN TOÀN — Chỉ truyền secrets cần thiết
- uses: trusted-action@v4
  with:
    token: ${{ secrets.SPECIFIC_TOKEN }}
```

### Lỗi 2: Pull request từ fork không có secrets (là thiết kế đúng)

```yaml
# Secrets KHÔNG được truyền vào workflow được kích hoạt bởi fork PRs
# Đây là tính năng bảo mật, không phải bug

# Nếu cần chạy tests cần secrets cho fork PRs:
on:
  pull_request_target:  # CẢNH BÁO: Chạy code từ PR trong context của base repo
```

### Lỗi 3: Secret hiển thị trong outputs

```yaml
# ❌ Secret bị lộ trong step output
- name: Set output with secret
  run: echo "::set-output name=token::${{ secrets.MY_TOKEN }}"

# ✅ Đặt output là masked
- name: Safe output
  run: |
    echo "token=${{ secrets.MY_TOKEN }}" >> $GITHUB_OUTPUT
    echo "::add-mask::${{ secrets.MY_TOKEN }}"  # Mask giá trị trong logs
```

### Lỗi 4: Không kiểm tra secret tồn tại trước khi dùng

```yaml
# ✅ Kiểm tra secret có giá trị không
- name: Verify secrets exist
  run: |
    if [ -z "${{ secrets.MY_SECRET }}" ]; then
      echo "Error: MY_SECRET is not set"
      exit 1
    fi
    echo "Secret is available (length: ${#MY_SECRET})"
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
```

---

## Câu Hỏi Phỏng Vấn

### Q: Tại sao bạn nên dùng environment secrets thay vì repository secrets cho production?

**Trả lời:**
Environment secrets cung cấp thêm một lớp kiểm soát:
1. **Protection Rules** — Có thể yêu cầu manual approval trước khi chạy
2. **Branch restrictions** — Chỉ main/release branch mới được deploy production
3. **Isolation** (Cô Lập) — Secrets của staging và production hoàn toàn tách biệt
4. **Audit trail** — Dễ theo dõi ai, khi nào truy cập production

### Q: GITHUB_TOKEN có những hạn chế gì?

**Trả lời:**
- Không thể trigger workflow mới từ workflow khác (tránh loop)
- Hết hạn sau khi workflow job kết thúc
- Scope giới hạn trong repository hiện tại (không cross-repo)
- Không thể dùng cho operations cần GitHub App permissions đặc biệt
- **Giải pháp:** Dùng Personal Access Token (PAT) hoặc GitHub App token cho các trường hợp đặc biệt

### Q: Làm thế nào để xử lý secrets trong matrix jobs?

```yaml
# Secrets hoạt động bình thường trong matrix
jobs:
  test:
    strategy:
      matrix:
        env: [staging, production]
    environment: ${{ matrix.env }}  # Mỗi job dùng environment secrets riêng
    steps:
      - run: echo "Running in ${{ matrix.env }}"
        env:
          DB_URL: ${{ secrets.DATABASE_URL }}  # Khác nhau mỗi environment
```

---

**Điều Hướng:**
- ← [README.md](README.md)
- → [2-variables.md](2-variables.md)

**Cập Nhật Lần Cuối:** 2026-05-11
