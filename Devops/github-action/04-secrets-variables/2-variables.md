# 📋 Variables — Biến Trong GitHub Actions

> Hướng dẫn toàn diện về Variables (Biến) trong GitHub Actions — phân biệt với Secrets, phạm vi ứng dụng, cách cấu hình và các pattern phổ biến.

---

## 📚 Mục Lục

1. [Variables Là Gì?](#variables-là-gì)
2. [Ba Loại Variables](#ba-loại-variables)
3. [Configuration Variables](#configuration-variables)
4. [Environment Variables](#environment-variables)
5. [Default Environment Variables](#default-environment-variables)
6. [Phạm Vi Variables](#phạm-vi-variables)
7. [Dynamic Variables](#dynamic-variables)
8. [Variables vs Secrets](#variables-vs-secrets)
9. [Patterns Thực Tế](#patterns-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Variables Là Gì?

Variables (Biến) trong GitHub Actions là cơ chế lưu trữ các giá trị cấu hình không nhạy cảm để tái sử dụng trong nhiều workflows. Khác với Secrets, Variables:
- **Không được mã hóa** (not encrypted at rest)
- **Có thể đọc lại** qua GitHub UI
- **Hiển thị trong logs** (không bị mask tự động)
- Phù hợp cho **cấu hình môi trường**, không phải credentials

### Khi Nào Dùng Variables?

```
Variables → App configs, URLs, version numbers, feature flags
Secrets   → Passwords, API keys, tokens, private keys
```

---

## Ba Loại Variables

```
┌────────────────────────────────────────────────────────────┐
│               GitHub Actions Variables                     │
│                                                            │
│  1. Configuration Variables (Biến Cấu Hình)               │
│     → Set trong GitHub UI hoặc API                        │
│     → Truy cập qua ${{ vars.MY_VAR }}                     │
│                                                            │
│  2. Environment Variables (Biến Môi Trường)               │
│     → Set trực tiếp trong workflow YAML                   │
│     → Truy cập qua $MY_VAR hoặc ${{ env.MY_VAR }}        │
│                                                            │
│  3. Default Variables (Biến Mặc Định)                     │
│     → GitHub tự động cung cấp (GITHUB_SHA, GITHUB_REF...) │
│     → Không cần khai báo                                  │
└────────────────────────────────────────────────────────────┘
```

---

## Configuration Variables

Configuration Variables (Biến Cấu Hình) được quản lý trong GitHub Settings — tương tự Secrets nhưng không mã hóa.

### Tạo Configuration Variable Qua UI

```
Repository → Settings → Secrets and variables → Actions → Variables tab → New repository variable
```

### Tạo Qua GitHub CLI

```bash
# Tạo repository variable
gh variable set APP_ENV --body "production"
gh variable set BASE_URL --body "https://api.example.com"
gh variable set MAX_RETRIES --body "3"

# Liệt kê variables
gh variable list

# Xóa variable
gh variable delete APP_ENV

# Organization variable
gh variable set SHARED_CONFIG \
  --org my-organization \
  --visibility all \
  --body "shared-value"
```

### Truy Cập Trong Workflow

```yaml
name: Configuration Variables Demo

on: push

env:
  # Dùng configuration variable ở workflow level
  BASE_URL: ${{ vars.BASE_URL }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Print configuration
        run: |
          echo "Environment: ${{ vars.APP_ENV }}"
          echo "Base URL: ${{ vars.BASE_URL }}"
          echo "Max Retries: ${{ vars.MAX_RETRIES }}"

      - name: Deploy with config
        run: |
          ./deploy.sh \
            --env "${{ vars.APP_ENV }}" \
            --url "${{ vars.BASE_URL }}" \
            --retries "${{ vars.MAX_RETRIES }}"
```

---

## Environment Variables

Environment Variables (Biến Môi Trường) được khai báo trực tiếp trong workflow YAML với từ khóa `env`.

### Các Cấp Độ Khai Báo

```yaml
name: Environment Variables Demo

# Cấp 1: Workflow level — áp dụng cho TẤT CẢ jobs và steps
env:
  NODE_ENV: production
  APP_VERSION: "1.0.0"

jobs:
  build:
    runs-on: ubuntu-latest
    
    # Cấp 2: Job level — áp dụng cho tất cả steps trong job này
    env:
      BUILD_DIR: ./dist
      LOG_LEVEL: info
    
    steps:
      - name: Step with env
        # Cấp 3: Step level — chỉ áp dụng cho step này
        env:
          STEP_SPECIFIC: "only for this step"
        run: |
          echo "NODE_ENV: $NODE_ENV"        # Từ workflow level
          echo "BUILD_DIR: $BUILD_DIR"      # Từ job level
          echo "STEP_SPECIFIC: $STEP_SPECIFIC"  # Từ step level
```

### Thứ Tự Ưu Tiên (Override Order)

```
Step env  >  Job env  >  Workflow env  >  Default env
(Cao nhất)                               (Thấp nhất)
```

### Ví Dụ Thực Tế: Node.js CI

```yaml
name: Node.js CI

on:
  push:
    branches: [main, develop]
  pull_request:

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    env:
      CI: true
      JEST_JUNIT_OUTPUT_DIR: ./test-results
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ env.NODE_VERSION }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      
      - run: npm ci
      - run: npm test
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://localhost:5432/testdb

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build \
            --tag "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" \
            --build-arg NODE_VERSION="${{ env.NODE_VERSION }}" \
            .
```

---

## Default Environment Variables

GitHub Actions tự động cung cấp nhiều biến môi trường hữu ích.

### Biến Về Repository và Branch

| Biến | Ví Dụ Giá Trị | Mô Tả |
|---|---|---|
| `GITHUB_REPOSITORY` | `owner/repo-name` | Tên đầy đủ của repository |
| `GITHUB_REPOSITORY_OWNER` | `owner` | Chủ sở hữu repository |
| `GITHUB_REF` | `refs/heads/main` | Git ref đầy đủ |
| `GITHUB_REF_NAME` | `main` | Tên branch hoặc tag |
| `GITHUB_REF_TYPE` | `branch` hoặc `tag` | Loại ref |
| `GITHUB_SHA` | `ffac537e...` | SHA của commit kích hoạt workflow |
| `GITHUB_HEAD_REF` | `feature/my-branch` | Source branch của PR |
| `GITHUB_BASE_REF` | `main` | Target branch của PR |

### Biến Về Workflow

| Biến | Ví Dụ Giá Trị | Mô Tả |
|---|---|---|
| `GITHUB_WORKFLOW` | `CI Pipeline` | Tên workflow |
| `GITHUB_WORKFLOW_REF` | `owner/repo/.github/workflows/ci.yml@refs/...` | Đường dẫn file workflow |
| `GITHUB_RUN_ID` | `1658821493` | ID duy nhất của workflow run |
| `GITHUB_RUN_NUMBER` | `42` | Số thứ tự của run trong workflow này |
| `GITHUB_RUN_ATTEMPT` | `1` | Lần thử thứ mấy (nếu re-run) |
| `GITHUB_JOB` | `build` | Tên của job hiện tại |
| `GITHUB_STEP_SUMMARY` | `/path/to/step_summary` | File để ghi step summary |

### Biến Về Event

| Biến | Ví Dụ Giá Trị | Mô Tả |
|---|---|---|
| `GITHUB_EVENT_NAME` | `push`, `pull_request` | Tên event kích hoạt |
| `GITHUB_EVENT_PATH` | `/home/runner/work/_temp/...` | Đường dẫn đến event payload JSON |
| `GITHUB_ACTOR` | `username` | Người kích hoạt workflow |
| `GITHUB_TRIGGERING_ACTOR` | `username` | Người kích hoạt gần nhất (kể cả re-run) |

### Biến Về Runner

| Biến | Ví Dụ Giá Trị | Mô Tả |
|---|---|---|
| `RUNNER_OS` | `Linux`, `Windows`, `macOS` | Hệ điều hành runner |
| `RUNNER_ARCH` | `X64`, `ARM64` | Kiến trúc CPU |
| `RUNNER_NAME` | `Hosted Agent` | Tên runner |
| `RUNNER_TEMP` | `/tmp` | Thư mục tạm thời (bị xóa sau job) |
| `RUNNER_TOOL_CACHE` | `/opt/hostedtoolcache` | Cache cho tools như Node, Python |

### Biến Về Paths

| Biến | Mô Tả |
|---|---|
| `GITHUB_WORKSPACE` | Thư mục checkout của repo |
| `GITHUB_PATH` | File để thêm vào PATH |
| `GITHUB_ENV` | File để set environment variables |
| `GITHUB_OUTPUT` | File để set step outputs |

### Sử Dụng Default Variables

```yaml
steps:
  - name: Print context information
    run: |
      echo "Repository: $GITHUB_REPOSITORY"
      echo "Branch: $GITHUB_REF_NAME"
      echo "Commit: $GITHUB_SHA"
      echo "Triggered by: $GITHUB_ACTOR"
      echo "Event: $GITHUB_EVENT_NAME"
      echo "Run ID: $GITHUB_RUN_ID"
      echo "Runner OS: $RUNNER_OS"

  - name: Create version tag
    run: |
      VERSION="${GITHUB_REF_NAME}-${GITHUB_SHA:0:7}"
      echo "Version: $VERSION"
      docker build -t myapp:$VERSION .
```

---

## Phạm Vi Variables

### Configuration Variables — Phạm Vi Theo Cấp

```
Organization Variables
    ├── Chia sẻ qua nhiều repos
    ├── ${{ vars.ORG_SHARED_VAR }}
    └── Visibility: all / private / selected repos

Repository Variables
    ├── Dành riêng cho một repo
    ├── ${{ vars.REPO_VAR }}
    └── Override organization variables cùng tên

Environment Variables (Configuration type)
    ├── Gắn với environment cụ thể (staging, production)
    ├── ${{ vars.ENV_SPECIFIC_VAR }}
    └── Override repo và org variables cùng tên
```

### Ví Dụ Hierarchy (Thứ Bậc)

```yaml
# Giả sử:
# Organization variable:  BASE_URL = "https://api.org.com"
# Repository variable:    BASE_URL = "https://api.myapp.com"  (override)
# Environment (prod) var: BASE_URL = "https://api.prod.com"   (override lại)

jobs:
  deploy:
    environment: production
    steps:
      - run: echo "${{ vars.BASE_URL }}"
        # Output: https://api.prod.com (environment value thắng)
```

---

## Dynamic Variables

Dynamic Variables (Biến Động) là biến được tạo trong quá trình chạy workflow.

### Set Variable Trong Script

```yaml
steps:
  - name: Calculate version
    run: |
      VERSION="1.0.$(date +%Y%m%d%H%M%S)"
      echo "APP_VERSION=$VERSION" >> $GITHUB_ENV
      echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_ENV

  - name: Use the dynamic variable
    run: |
      echo "Version: $APP_VERSION"
      echo "Build date: $BUILD_DATE"
```

### Set Variable Từ Command Output

```yaml
steps:
  - name: Get latest tag
    run: |
      LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
      echo "LATEST_TAG=$LATEST_TAG" >> $GITHUB_ENV

  - name: Use latest tag
    run: echo "Latest tag is $LATEST_TAG"
```

### Conditional Variables

```yaml
steps:
  - name: Set environment name
    run: |
      if [ "$GITHUB_REF_NAME" == "main" ]; then
        echo "DEPLOY_ENV=production" >> $GITHUB_ENV
      elif [ "$GITHUB_REF_NAME" == "develop" ]; then
        echo "DEPLOY_ENV=staging" >> $GITHUB_ENV
      else
        echo "DEPLOY_ENV=development" >> $GITHUB_ENV
      fi

  - name: Deploy
    run: ./deploy.sh $DEPLOY_ENV
```

### Step Outputs Làm Variables

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      tag: ${{ steps.version.outputs.tag }}
    steps:
      - name: Calculate version
        id: version
        run: |
          VERSION=$(cat package.json | jq -r .version)
          TAG="v$VERSION-$(git rev-parse --short HEAD)"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  deploy:
    needs: prepare
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with version
        run: |
          echo "Deploying version: ${{ needs.prepare.outputs.version }}"
          echo "Using tag: ${{ needs.prepare.outputs.tag }}"
```

---

## Variables vs Secrets

| Tiêu Chí | Variables (`vars.*`) | Secrets (`secrets.*`) |
|---|---|---|
| **Mã hóa** | ❌ Không | ✅ Có |
| **Hiển thị trong UI** | ✅ Có thể đọc lại | ❌ Không (chỉ update) |
| **Masked trong logs** | ❌ Không tự động | ✅ Luôn luôn |
| **Dùng cho** | App config, URLs, flags | Passwords, keys, tokens |
| **Có thể in ra logs** | Được (cẩn thận) | Không nên |
| **Override** | Env > Repo > Org | Env > Repo > Org |
| **Truy cập** | `${{ vars.NAME }}` | `${{ secrets.NAME }}` |

### Quy Tắc Quyết Định Nhanh

```
Câu hỏi: "Nếu người khác đọc được giá trị này, có vấn đề không?"

Có  → Dùng Secrets
Không → Dùng Variables
```

---

## Patterns Thực Tế

### Pattern 1: Feature Flags (Cờ Tính Năng)

```yaml
# Repository variable: FEATURE_NEW_UI = "true"
# Repository variable: FEATURE_ANALYTICS = "false"

steps:
  - name: Run with feature flags
    run: |
      if [ "${{ vars.FEATURE_NEW_UI }}" == "true" ]; then
        echo "Running new UI tests"
        npm run test:new-ui
      fi
    
  - name: Deploy with feature flags
    run: |
      FEATURES='{"newUI": ${{ vars.FEATURE_NEW_UI }}, "analytics": ${{ vars.FEATURE_ANALYTICS }}}'
      ./deploy.sh --features "$FEATURES"
```

### Pattern 2: Multi-Environment Config

```yaml
# Staging environment variable:   API_URL = "https://api.staging.com"
# Production environment variable: API_URL = "https://api.production.com"

jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: ./deploy.sh --api-url "${{ vars.API_URL }}"

  deploy-production:
    environment: production
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: ./deploy.sh --api-url "${{ vars.API_URL }}"  # Khác giá trị!
```

### Pattern 3: Versioning

```yaml
# Repository variable: APP_VERSION = "2.5.0"
# Repository variable: MIN_COVERAGE = "80"

steps:
  - name: Build with version
    run: |
      docker build \
        --label "version=${{ vars.APP_VERSION }}" \
        --label "git-sha=${{ github.sha }}" \
        -t myapp:${{ vars.APP_VERSION }} \
        .

  - name: Check coverage threshold
    run: |
      COVERAGE=$(./get-coverage.sh)
      if [ "$COVERAGE" -lt "${{ vars.MIN_COVERAGE }}" ]; then
        echo "Coverage $COVERAGE% is below minimum ${{ vars.MIN_COVERAGE }}%"
        exit 1
      fi
```

### Pattern 4: Shared Organization Config

```yaml
# Organization variables:
#   ORG_REGISTRY = "ghcr.io/my-org"
#   ORG_REGION = "us-east-1"
#   ORG_TEAM_EMAIL = "devops@company.com"

steps:
  - name: Push to organization registry
    run: |
      docker tag myapp:latest "${{ vars.ORG_REGISTRY }}/myapp:latest"
      docker push "${{ vars.ORG_REGISTRY }}/myapp:latest"

  - name: Notify team on failure
    if: failure()
    uses: dawidd6/action-send-mail@v3
    with:
      to: ${{ vars.ORG_TEAM_EMAIL }}
      subject: "Build Failed: ${{ github.repository }}"
```

### Pattern 5: Combining Variables và Contexts

```yaml
steps:
  - name: Build Docker image with full tag
    run: |
      # Kết hợp variable và context (github.*)
      FULL_TAG="${{ vars.REGISTRY }}/${{ github.repository }}:${{ github.sha }}"
      SHORT_TAG="${{ vars.REGISTRY }}/${{ github.repository }}:${{ vars.APP_VERSION }}"
      
      docker build -t "$FULL_TAG" -t "$SHORT_TAG" .
      docker push "$FULL_TAG"
      docker push "$SHORT_TAG"
```

---

## Câu Hỏi Phỏng Vấn

### Q: Khi nào bạn dùng `env:` trong workflow thay vì configuration variables (`vars.*`)?

**Trả lời:**
- `env:` trong YAML: Khi giá trị phụ thuộc vào context của workflow run (SHA, branch name, dynamic values), hoặc khi tính đơn giản quan trọng hơn tái sử dụng
- `vars.*`: Khi giá trị cần quản lý tập trung qua UI, chia sẻ across workflows, hoặc khác nhau giữa các environments mà không sửa YAML

### Q: Biến đặt trong `$GITHUB_ENV` có tồn tại qua các jobs không?

**Trả lời:** Không. `$GITHUB_ENV` chỉ tồn tại trong phạm vi job hiện tại. Để truyền giá trị qua jobs, phải dùng job outputs (`$GITHUB_OUTPUT`) và `needs.<job>.outputs`.

```yaml
jobs:
  job1:
    outputs:
      my_var: ${{ steps.set.outputs.my_var }}
    steps:
      - id: set
        run: echo "my_var=hello" >> $GITHUB_OUTPUT

  job2:
    needs: job1
    steps:
      - run: echo "${{ needs.job1.outputs.my_var }}"
```

### Q: Có thể dùng variables trong `if:` conditions không?

```yaml
# ✅ CÓ THỂ — Variables có thể dùng trong expressions
jobs:
  deploy:
    if: ${{ vars.DEPLOY_ENABLED == 'true' }}
    steps:
      - name: Conditional step
        if: ${{ vars.FEATURE_FLAG == 'enabled' }}
        run: echo "Feature is enabled"
```

---

**Điều Hướng:**
- ← [1-secrets-management.md](1-secrets-management.md)
- → [3-oidc.md](3-oidc.md)

**Cập Nhật Lần Cuối:** 2026-05-11
