# 🕵️ Secret Scanning — Phát Hiện Secrets Bị Lộ

> Secret Scanning — Quét Bí Mật là cơ chế tự động phát hiện các thông tin nhạy cảm (API keys, passwords, tokens, certificates) bị commit nhầm vào code repository, và ngăn chặn chúng trước khi đến tay kẻ tấn công.

---

## 📚 Mục Lục

1. [Tại Sao Secrets Bị Lộ Là Nguy Hiểm?](#tại-sao-secrets-bị-lộ-là-nguy-hiểm)
2. [GitHub Secret Scanning Tích Hợp Sẵn](#github-secret-scanning-tích-hợp-sẵn)
3. [Push Protection — Ngăn Chặn Trước Khi Push](#push-protection)
4. [Gitleaks — Quét Tích Hợp Vào CI Pipeline](#gitleaks)
5. [TruffleHog — Quét Lịch Sử Git](#trufflehog)
6. [detect-secrets — Pre-commit Hook](#detect-secrets)
7. [Xử Lý Khi Phát Hiện Secret Bị Lộ](#xử-lý-khi-phát-hiện-secret-bị-lộ)
8. [Ngăn Chặn Secrets Bị Lộ Từ Đầu](#ngăn-chặn-secrets-bị-lộ-từ-đầu)

---

## Tại Sao Secrets Bị Lộ Là Nguy Hiểm?

### Số Liệu Thực Tế

```
- Hàng triệu secrets được commit lên GitHub mỗi năm
- Thời gian trung bình từ khi lộ đến khi bị khai thác: < 4 giây (đối với bot tự động quét)
- 80% breaches liên quan đến credentials bị lộ
- Git history giữ lại secrets ngay cả sau khi xóa file
```

### Các Loại Secrets Thường Bị Lộ

```
Token & API Keys:
  - AWS Access Key (AKIA...)
  - GitHub Personal Access Token (ghp_...)
  - Stripe API Key (sk_live_...)
  - OpenAI API Key (sk-...)
  - Google API Key (AIza...)
  - Slack Bot Token (xoxb-...)

Credentials:
  - Database passwords trong connection strings
  - SSH private keys
  - SSL/TLS certificates và private keys
  - JWT signing secrets

Cloud Credentials:
  - Google Cloud service account JSON
  - Azure client secrets
  - Kubernetes kubeconfig với credentials
```

### Tại Sao Xóa File Không Đủ

```bash
# Sai lầm phổ biến: nghĩ rằng xóa file là đủ
git add .env          # Commit chứa secrets
git commit -m "add config"
git push

# Sau đó xóa file
git rm .env
git commit -m "remove secrets"
git push

# NHƯNG: Secret vẫn còn trong git history!
git log --all --full-history -- .env  # Vẫn thấy!
git show COMMIT_SHA:.env              # Vẫn đọc được!
```

---

## GitHub Secret Scanning Tích Hợp Sẵn

### Bật Secret Scanning

```
Repository Settings
  → Security
    → Code security and analysis
      → Secret scanning: Enable
      → Push protection: Enable (khuyến nghị mạnh)
```

### Cách Hoạt Động

```
1. GitHub quét tất cả commits và blob trong repository
2. So sánh với hơn 200+ pattern của các provider (AWS, Google, Stripe, v.v.)
3. Nếu phát hiện:
   → Tạo alert trong Security tab
   → Notify repository admin qua email
   → Với Push Protection: từ chối push
   → Với partner program: notify trực tiếp provider (AWS, GitHub, v.v.)
```

### Xem và Xử Lý Alerts

```bash
# Qua GitHub CLI
gh secret-scanning list --repo owner/repo

# Xem chi tiết alert
gh secret-scanning view 123 --repo owner/repo

# Đánh dấu là false positive (không phải secrets thật)
gh secret-scanning resolve 123 --repo owner/repo --reason false_positive

# Đánh dấu là đã xử lý
gh secret-scanning resolve 123 --repo owner/repo --reason revoked
```

### Custom Patterns (Enterprise)

```yaml
# Thêm pattern tùy chỉnh cho secrets nội bộ
# GitHub Enterprise hoặc Advanced Security

# Ví dụ: phát hiện internal API token format
Pattern Name: Internal API Token
Pattern: MYCOMPANY-[A-Z0-9]{32}
Test string: MYCOMPANY-ABCD1234567890123456789012345678
```

---

## Push Protection

### Push Protection Là Gì?

**Push Protection** — Bảo Vệ Push là tính năng tự động từ chối push nếu phát hiện secret trong commit, trước khi code lên remote repository.

```
Luồng Với Push Protection:
  git push
    → GitHub quét diff
    → Phát hiện AWS key trong file config.js
    → TỪ CHỐI push với thông báo:

  remote: error: GH013: Repository rule violations found for refs/heads/main.
  remote: - GITHUB PUSH PROTECTION
  remote:   ————————————————————————————————————————
  remote:     Resolve the following secrets before pushing:
  remote:
  remote:     (aws-access-key-id) AWS Access Key ID
  remote:     ————————————————————————————————————————
```

### Bypass Push Protection (Khi Cần Thiết)

```bash
# Trường hợp hợp lệ: test fixtures, fake tokens trong documentation

# 1. Qua GitHub web: click "Bypass protection" và chọn lý do
# 2. Qua git: thêm query param
git push origin main

# Hoặc sử dụng flag --no-verify (không khuyến nghị)
# Bypass này được log lại trong audit log
```

---

## Gitleaks — Quét Tích Hợp Vào CI Pipeline

### Gitleaks Là Gì?

**Gitleaks** là công cụ mã nguồn mở, nhanh, dùng để phát hiện secrets đã commit vào git repository. Có thể chạy như pre-commit hook hoặc trong CI pipeline.

### Tích Hợp Gitleaks Vào GitHub Actions

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scanning

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  security-events: write  # Để upload SARIF

jobs:
  gitleaks:
    name: Gitleaks Secret Scan
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0  # Cần full history để scan toàn bộ

      - name: Chạy Gitleaks
        uses: gitleaks/gitleaks-action@ff98106e4c7b2bc287b24a84a2f1d39c2e9a8e00 # v2.3.9
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # Cần cho org scans

      - name: Upload SARIF nếu có findings
        if: failure()
        uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        with:
          sarif_file: results.sarif
```

### Cấu Hình Gitleaks Custom Rules

```toml
# .gitleaks.toml
title = "Gitleaks Custom Config"

[[rules]]
id = "internal-api-key"
description = "Internal API Key"
regex = '''MYCOMPANY-[A-Z0-9]{32}'''
tags = ["internal", "api-key"]

[[rules]]
id = "database-url"
description = "Database Connection String With Password"
regex = '''(?i)(mysql|postgres|mongodb):\/\/[^:]+:[^@]+@'''
tags = ["database", "credentials"]

[allowlist]
description = "Allowlist cho test fixtures"
paths = [
  '''test/fixtures/.*''',
  '''docs/examples/.*'''
]
regexes = [
  '''EXAMPLE_KEY_FOR_DOCS'''  # Từ khóa fake trong documentation
]
commits = [
  "abc123def456"  # Commit cụ thể đã được review và chấp nhận
]
```

---

## TruffleHog — Quét Lịch Sử Git

### TruffleHog — Tìm Secrets Trong Toàn Bộ Git History

```yaml
name: TruffleHog Secret Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Cần cho verified results

jobs:
  trufflehog:
    name: TruffleHog Scan
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0  # Full git history

      - name: Quét với TruffleHog
        uses: trufflesecurity/trufflehog@d77d64fabd2cba63bbddba8dc8d5e77c7a779b9b # v3.88.26
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified  # Chỉ báo cáo secrets đã xác minh thật

      - name: Quét toàn bộ history (chạy một lần khi setup)
        if: github.event_name == 'workflow_dispatch'
        uses: trufflesecurity/trufflehog@d77d64fabd2cba63bbddba8dc8d5e77c7a779b9b # v3.88.26
        with:
          path: ./
          extra_args: --only-verified
```

---

## detect-secrets — Pre-commit Hook

### Ngăn Chặn Tại Máy Developer (Pre-commit)

```bash
# Cài đặt detect-secrets
pip install detect-secrets

# Tạo baseline (danh sách secrets đã biết và được phép)
detect-secrets scan > .secrets.baseline

# Cài đặt pre-commit hook
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        exclude: package.lock.json
```

```bash
# Cài đặt hooks
pre-commit install

# Kiểm tra thủ công
pre-commit run detect-secrets --all-files
```

### Cập Nhật Baseline Khi Có False Positives

```bash
# Kiểm tra và update baseline
detect-secrets scan --update .secrets.baseline

# Audit baseline — review từng potential secret
detect-secrets audit .secrets.baseline
```

---

## Xử Lý Khi Phát Hiện Secret Bị Lộ

### Quy Trình Xử Lý Khẩn Cấp

```
🚨 BƯỚC 1: Thu Hồi Ngay Lập Tức (Trong Vài Phút)
  → Revoke/rotate credential BỊ LỘ NGAY LẬP TỨC
  → Không chờ "điều tra xong rồi thu hồi" — TỐC ĐỘ LÀ QUAN TRỌNG NHẤT

🔍 BƯỚC 2: Đánh Giá Mức Độ Ảnh Hưởng
  → Secret đó có quyền truy cập gì?
  → Đã lộ bao lâu? (git log --follow)
  → Ai có thể đã thấy? (public repo vs private repo)
  → Có log nào bất thường không?

🧹 BƯỚC 3: Làm Sạch Git History (Nếu Cần)
  → Dùng git-filter-repo hoặc BFG Repo Cleaner
  → KHÔNG dùng git filter-branch (chậm và nguy hiểm)

📢 BƯỚC 4: Thông Báo Nếu Cần
  → Thông báo team
  → Nếu có data breach → xem xét thông báo người dùng theo luật
  → Cập nhật postmortem
```

### Xóa Secret Khỏi Git History

```bash
# Cách 1: Dùng git-filter-repo (khuyến nghị)
pip install git-filter-repo

# Xóa file .env khỏi toàn bộ history
git filter-repo --path .env --invert-paths

# Xóa chuỗi cụ thể khỏi tất cả files trong history
git filter-repo --replace-text replacements.txt
# replacements.txt:
# ACTUAL_SECRET_VALUE==>REDACTED

# Cách 2: BFG Repo Cleaner (nhanh hơn cho repo lớn)
# Download BFG từ rtyley.github.io/bfg-repo-cleaner
java -jar bfg.jar --replace-text passwords.txt my-repo.git

# SAU KHI CLEAN: Force push lên tất cả branches
git push origin --force --all
git push origin --force --tags

# ⚠️ QUAN TRỌNG: Thông báo tất cả collaborators phải:
# git fetch --all
# git reset --hard origin/main
```

### Verify Secret Đã Bị Thu Hồi

```bash
# AWS: Kiểm tra xem key còn active không
aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE

# GitHub PAT: Kiểm tra và revoke
# github.com/settings/tokens

# Stripe: Kiểm tra trong dashboard
# dashboard.stripe.com/apikeys

# Tổng quát: Thử dùng secret đã thu hồi
# Nếu bị từ chối → đã thu hồi thành công
```

---

## Ngăn Chặn Secrets Bị Lộ Từ Đầu

### .gitignore Đúng Cách

```gitignore
# .gitignore — Luôn ignore các file nhạy cảm

# Environment files
.env
.env.local
.env.*.local
.env.production
.env.staging
*.env

# Credentials và keys
*.pem
*.key
*.p12
*.pfx
id_rsa
id_ed25519
*.json  # Cẩn thận — Google service account là .json

# Config files chứa credentials
config/secrets.yml
config/database.yml  # Rails
application-production.properties  # Spring Boot
appsettings.Production.json  # .NET

# Cloud credentials
.aws/credentials
.azure/
gcloud/

# Không bao giờ commit
*.secret
*credentials*
*password*
```

### Template Environment File

```bash
# Thay vì .env, commit .env.example (không có giá trị thật)
# .env.example
DATABASE_URL=postgres://user:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379
AWS_ACCESS_KEY_ID=your_aws_access_key_here
AWS_SECRET_ACCESS_KEY=your_aws_secret_key_here
STRIPE_SECRET_KEY=sk_test_your_stripe_key_here
JWT_SECRET=your_jwt_secret_here_minimum_32_chars

# Developer clone repo → cp .env.example .env → điền giá trị thật vào .env
```

### Workflow Kiểm Tra Không Có Secrets Hardcoded

```yaml
name: Secret Hygiene Check

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  check-secrets:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0

      - name: Gitleaks scan on PR diff
        uses: gitleaks/gitleaks-action@ff98106e4c7b2bc287b24a84a2f1d39c2e9a8e00 # v2.3.9
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Kiểm tra .env files không bị commit
        run: |
          # Kiểm tra xem có .env files trong staged changes không
          if git diff --name-only origin/${{ github.base_ref }}...HEAD \
              | grep -qE '^\.env(\.|$)'; then
            echo "::error::.env file bị commit — không được phép!"
            exit 1
          fi

      - name: Kiểm tra không có AWS keys hardcoded
        run: |
          if git diff origin/${{ github.base_ref }}...HEAD \
              | grep -qE 'AKIA[0-9A-Z]{16}'; then
            echo "::error::Phát hiện AWS Access Key trong diff!"
            exit 1
          fi
```

---

## 📋 Checklist Phòng Chống Secret Leakage

### Cho Developer

- [ ] Cài đặt pre-commit hook với detect-secrets hoặc gitleaks
- [ ] Luôn thêm `.env` và credential files vào `.gitignore`
- [ ] Dùng password manager thay vì lưu credentials trong text files
- [ ] Review diff trước khi commit (`git diff --staged`)
- [ ] Không bao giờ paste secrets vào Slack, Teams, comment PR

### Cho Repository

- [ ] Bật GitHub Secret Scanning
- [ ] Bật Push Protection
- [ ] Thêm Gitleaks hoặc TruffleHog vào CI pipeline
- [ ] Cấu hình `.gitleaks.toml` với custom rules cho codebase
- [ ] Định kỳ chạy full history scan

### Cho Organization

- [ ] Bật secret scanning ở cấp organization
- [ ] Cấu hình custom patterns cho internal credentials
- [ ] Alert notification đến security team
- [ ] Training định kỳ cho developers về secret hygiene
- [ ] Rotation policy cho tất cả credentials

---

## 🔗 Xem Thêm

- [1-permissions.md](./1-permissions.md) — Kiểm soát quyền GITHUB_TOKEN
- [4-code-scanning.md](./4-code-scanning.md) — SAST/DAST scanning
- [6-security-hardening.md](./6-security-hardening.md) — Hardening checklist
- [04-secrets-variables/1-secrets-management.md](../04-secrets-variables/1-secrets-management.md) — Quản lý secrets đúng cách

---

**Cập Nhật Lần Cuối:** 2026-05-12
