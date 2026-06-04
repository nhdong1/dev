# Bài Tập Thực Hành GitHub Actions

> 15 bài tập hands-on (Thực Hành Trực Tiếp) có đáp án mẫu — từ cơ bản đến nâng cao. Mỗi bài tập bao gồm: mô tả, requirements, hints và solution (Giải Pháp) đầy đủ.

---

## 🛠️ Cách Thực Hành Hiệu Quả

**Chuẩn bị:**

```bash
# Tạo repository thực hành trên GitHub
# Clone về máy
git clone https://github.com/your-username/github-actions-practice
cd github-actions-practice

# Tạo thư mục workflows
mkdir -p .github/workflows

# Cài act để test locally (tùy chọn)
# https://github.com/nektos/act
brew install act  # macOS
```

**Quy trình cho mỗi bài:**
1. Đọc mô tả và requirements
2. Tự viết workflow không nhìn đáp án (15–30 phút)
3. Test bằng cách push lên GitHub
4. So sánh với đáp án mẫu
5. Hiểu tại sao đáp án làm như vậy

---

## 📚 Danh Sách Bài Tập

| # | Tên | Cấp Độ | Chủ Đề | Thời Gian |
|---|---|---|---|---|
| 1 | Hello World Workflow | ⭐ Cơ Bản | Events, Steps | 15 phút |
| 2 | CI Pipeline Node.js | ⭐⭐ Cơ Bản | CI, Testing | 30 phút |
| 3 | Multi-version Testing | ⭐⭐ Cơ Bản | Matrix Strategy | 20 phút |
| 4 | Cache Dependencies | ⭐⭐ Trung Bình | Caching | 25 phút |
| 5 | Upload Build Artifacts | ⭐⭐ Trung Bình | Artifacts | 20 phút |
| 6 | Share Data Between Jobs | ⭐⭐ Trung Bình | Job Outputs | 25 phút |
| 7 | Environment Deployments | ⭐⭐⭐ Trung Bình | Environments, CD | 40 phút |
| 8 | Reusable Workflow | ⭐⭐⭐ Trung Bình | Reusable | 35 phút |
| 9 | Docker Build & Push | ⭐⭐⭐ Trung Bình | Docker, Registry | 30 phút |
| 10 | Security Scanning | ⭐⭐⭐ Trung Bình | Security | 25 phút |
| 11 | Slack Notifications | ⭐⭐⭐ Trung Bình | Notifications | 20 phút |
| 12 | OIDC with AWS | ⭐⭐⭐⭐ Nâng Cao | OIDC, AWS | 45 phút |
| 13 | Custom Composite Action | ⭐⭐⭐⭐ Nâng Cao | Custom Actions | 40 phút |
| 14 | Release Automation | ⭐⭐⭐⭐ Nâng Cao | Versioning, Release | 45 phút |
| 15 | Full CI/CD Pipeline | ⭐⭐⭐⭐⭐ Thách Thức | End-to-end | 2–3 giờ |

---

## Bài Tập 1: Hello World Workflow

**Cấp độ:** ⭐ Cơ Bản | **Thời gian:** 15 phút

### Mô Tả

Tạo workflow đầu tiên in ra thông tin cơ bản về run context (Ngữ Cảnh Chạy).

### Requirements

- Trigger (Kích Hoạt) khi push lên bất kỳ branch nào
- In ra: tên branch, tên người commit, commit message, event type
- Workflow có ít nhất 2 steps

### Hints (Gợi Ý)

- Dùng `${{ github.ref_name }}` cho tên branch
- Dùng `${{ github.actor }}` cho tên người thực hiện
- Dùng `${{ github.event_name }}` cho loại event

### Đáp Án Mẫu

```yaml
# .github/workflows/hello-world.yml
name: Hello World

on: push

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Print run context
        run: |
          echo "=== Run Information ==="
          echo "Branch:         ${{ github.ref_name }}"
          echo "Triggered by:   ${{ github.actor }}"
          echo "Event:          ${{ github.event_name }}"
          echo "Commit SHA:     ${{ github.sha }}"
          echo "Workflow:       ${{ github.workflow }}"

      - name: Print commit message
        run: |
          echo "=== Latest Commit ==="
          echo "Message: ${{ github.event.head_commit.message }}"
          echo "Author:  ${{ github.event.head_commit.author.name }}"
          echo "Time:    ${{ github.event.head_commit.timestamp }}"
```

### Học Được Gì

- Cấu trúc cơ bản của file workflow
- Contexts (Ngữ Cảnh) `github.*` phổ biến
- Multi-step job

---

## Bài Tập 2: CI Pipeline Node.js

**Cấp độ:** ⭐⭐ Cơ Bản | **Thời gian:** 30 phút

### Mô Tả

Tạo CI pipeline đầy đủ cho Node.js application.

### Chuẩn Bị

```bash
# Tạo package.json đơn giản
cat > package.json << 'EOF'
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "test": "echo 'Tests passed!' && exit 0",
    "lint": "echo 'Lint passed!' && exit 0",
    "build": "echo 'Build completed!' && mkdir -p dist && echo '{}' > dist/app.js"
  }
}
EOF
```

### Requirements

- Trigger trên `push` và `pull_request` vào nhánh `main`
- Sử dụng Node.js version 20
- Chạy theo thứ tự: lint → test → build
- Nếu lint fail thì không chạy test
- Upload build output như artifact (Tệp Đầu Ra)

### Đáp Án Mẫu

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm run lint

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint                  # Chờ lint pass
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm test

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test                  # Chờ test pass
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: dist/
          retention-days: 7
```

---

## Bài Tập 3: Multi-version Testing với Matrix Strategy

**Cấp độ:** ⭐⭐ Cơ Bản | **Thời gian:** 20 phút

### Requirements

- Test trên Node.js 18, 20, 22
- Test trên Ubuntu và macOS
- Bỏ qua combination `macos + Node 18` (chi phí macOS cao)
- Hiển thị matrix combination nào đang chạy trong step name

### Đáp Án Mẫu

```yaml
# .github/workflows/matrix-test.yml
name: Matrix Tests

on: push

jobs:
  test:
    name: Test Node ${{ matrix.node }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false           # Không dừng tất cả nếu một combination fail
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, macos-latest]
        exclude:
          - os: macos-latest     # Loại bỏ macos + Node 18
            node: 18

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - name: Display environment
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Node version: $(node --version)"
          echo "npm version: $(npm --version)"

      - run: npm test
```

### Kết Quả Mong Đợi

5 jobs chạy song song: (Ubuntu + Node 18), (Ubuntu + Node 20), (Ubuntu + Node 22), (macOS + Node 20), (macOS + Node 22).

---

## Bài Tập 4: Cache Dependencies Hiệu Quả

**Cấp độ:** ⭐⭐ Trung Bình | **Thời gian:** 25 phút

### Requirements

- Cache `node_modules` với cache key dựa trên `package-lock.json`
- Có restore-keys (Khóa Khôi Phục) làm fallback
- In ra cache hit/miss status
- Đo thời gian install có và không có cache

### Đáp Án Mẫu

```yaml
# .github/workflows/cache-demo.yml
name: Cache Demo

on: push

jobs:
  with-cache:
    name: With Cache
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Cache node_modules
        id: cache
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Report cache status
        run: |
          if [ "${{ steps.cache.outputs.cache-hit }}" == "true" ]; then
            echo "✅ Cache HIT — skipping install"
          else
            echo "❌ Cache MISS — running npm ci"
          fi

      - name: Install dependencies (if cache miss)
        if: steps.cache.outputs.cache-hit != 'true'
        run: |
          echo "Installing dependencies..."
          time npm ci

      - name: Verify installation
        run: node --version && npm list --depth=0 2>/dev/null | head -5
```

**Lưu ý thực tế:** Với `setup-node@v4`, dùng tham số `cache: 'npm'` đơn giản hơn:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'            # Tự động cache npm, không cần actions/cache thủ công
```

---

## Bài Tập 5: Upload & Download Artifacts

**Cấp độ:** ⭐⭐ Trung Bình | **Thời gian:** 20 phút

### Requirements

- Job `build` tạo file `dist/app.js` và upload như artifact
- Job `test-artifact` download và verify artifact tồn tại
- Artifact giữ trong 3 ngày

### Đáp Án Mẫu

```yaml
# .github/workflows/artifacts-demo.yml
name: Artifacts Demo

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Create build output
        run: |
          mkdir -p dist
          echo "console.log('Hello from build!');" > dist/app.js
          echo "Build info: ${{ github.sha }}" > dist/build-info.txt

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-build-${{ github.run_number }}
          path: |
            dist/
            !dist/**/*.map      # Loại trừ source maps
          retention-days: 3
          if-no-files-found: error  # Fail nếu không có file nào

  verify:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: app-build-${{ github.run_number }}
          path: downloaded-dist/

      - name: Verify artifact contents
        run: |
          echo "=== Downloaded files ==="
          ls -la downloaded-dist/
          echo ""
          echo "=== app.js contents ==="
          cat downloaded-dist/app.js
          echo ""
          echo "=== build-info.txt ==="
          cat downloaded-dist/build-info.txt
```

---

## Bài Tập 6: Chia Sẻ Dữ Liệu Giữa Jobs Bằng Outputs

**Cấp độ:** ⭐⭐ Trung Bình | **Thời gian:** 25 phút

### Requirements

- Job `prepare` đọc version từ `package.json` và export như output
- Job `build` nhận version và dùng làm tag cho artifact
- Job `notify` đọc version và in ra "Đã build version X.Y.Z"

### Đáp Án Mẫu

```yaml
# .github/workflows/job-outputs.yml
name: Job Outputs Demo

on: push

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
      build-date: ${{ steps.get-date.outputs.date }}

    steps:
      - uses: actions/checkout@v4

      - name: Get version from package.json
        id: get-version
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Extracted version: $VERSION"

      - name: Get build date
        id: get-date
        run: |
          DATE=$(date -u +%Y%m%d)
          echo "date=$DATE" >> $GITHUB_OUTPUT

  build:
    runs-on: ubuntu-latest
    needs: prepare
    steps:
      - uses: actions/checkout@v4

      - name: Build with version tag
        run: |
          echo "Building version: ${{ needs.prepare.outputs.version }}"
          mkdir -p dist
          echo "v${{ needs.prepare.outputs.version }}" > dist/VERSION

      - uses: actions/upload-artifact@v4
        with:
          name: build-v${{ needs.prepare.outputs.version }}-${{ needs.prepare.outputs.build-date }}
          path: dist/

  notify:
    runs-on: ubuntu-latest
    needs: [prepare, build]
    steps:
      - name: Deployment summary
        run: |
          echo "✅ Build completed!"
          echo "Version: ${{ needs.prepare.outputs.version }}"
          echo "Date:    ${{ needs.prepare.outputs.build-date }}"
          echo "Commit:  ${{ github.sha }}"
```

---

## Bài Tập 7: CD Pipeline Với Environments

**Cấp độ:** ⭐⭐⭐ Trung Bình | **Thời gian:** 40 phút

### Chuẩn Bị Trên GitHub

Vào Settings → Environments → tạo 2 environments: `staging` và `production`.
Với `production`: bật "Required reviewers", thêm bản thân làm reviewer.

### Requirements

- Trigger khi push vào `main`
- Deploy lên staging tự động
- Deploy lên production chỉ sau khi có manual approval
- Mỗi job in ra environment URL

### Đáp Án Mẫu

```yaml
# .github/workflows/cd-environments.yml
name: CD with Environments

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Simulate deployment to staging
        run: |
          echo "🚀 Deploying to STAGING environment..."
          echo "Commit: ${{ github.sha }}"
          echo "Deployed by: ${{ github.actor }}"
          sleep 2  # Simulate deploy time
          echo "✅ Staging deployment complete!"

      - name: Run smoke tests
        run: |
          echo "🧪 Running smoke tests on staging..."
          # Trong thực tế: curl https://staging.example.com/health
          echo "✅ Smoke tests passed!"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://example.com

    steps:
      - uses: actions/checkout@v4

      - name: Simulate deployment to production
        run: |
          echo "🚀 Deploying to PRODUCTION environment..."
          echo "Commit: ${{ github.sha }}"
          echo "Approved and deployed by: ${{ github.actor }}"
          sleep 2
          echo "✅ Production deployment complete!"
```

### Kiểm Tra Kết Quả

- Push commit → staging deploy tự động
- Production job dừng lại → vào GitHub → click "Review deployments" → approve
- Production deploy tiếp tục

---

## Bài Tập 8: Reusable Workflow

**Cấp độ:** ⭐⭐⭐ Trung Bình | **Thời gian:** 35 phút

### Requirements

- Tạo reusable workflow nhận `app-name` và `node-version` làm inputs
- Nhận `npm-auth-token` làm secret (optional)
- Workflow thực hiện: checkout → setup Node → install → test → build
- Tạo caller workflow gọi reusable workflow

### Đáp Án Mẫu

**Reusable workflow — `.github/workflows/reusable-ci.yml`:**

```yaml
name: Reusable CI

on:
  workflow_call:
    inputs:
      app-name:
        description: 'Application name'
        required: true
        type: string
      node-version:
        description: 'Node.js version'
        required: false
        default: '20'
        type: string
      run-tests:
        description: 'Whether to run tests'
        required: false
        default: true
        type: boolean
    secrets:
      npm-auth-token:
        required: false
    outputs:
      build-status:
        description: 'Build result'
        value: ${{ jobs.ci.outputs.status }}

jobs:
  ci:
    name: CI for ${{ inputs.app-name }}
    runs-on: ubuntu-latest
    outputs:
      status: ${{ steps.build.outcome }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ inputs.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'

      - name: Configure npm registry
        if: secrets.npm-auth-token != ''
        run: |
          echo "//registry.npmjs.org/:_authToken=${{ secrets.npm-auth-token }}" > ~/.npmrc

      - run: npm ci

      - name: Run tests
        if: inputs.run-tests
        run: npm test

      - name: Build
        id: build
        run: npm run build
```

**Caller workflow — `.github/workflows/app-ci.yml`:**

```yaml
name: App CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  run-ci:
    uses: ./.github/workflows/reusable-ci.yml
    with:
      app-name: my-awesome-app
      node-version: '20'
      run-tests: true
    secrets:
      npm-auth-token: ${{ secrets.NPM_TOKEN }}

  use-output:
    needs: run-ci
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build status was ${{ needs.run-ci.outputs.build-status }}"
```

---

## Bài Tập 9: Docker Build và Push lên GHCR

**Cấp độ:** ⭐⭐⭐ Trung Bình | **Thời gian:** 30 phút

### Chuẩn Bị

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

### Requirements

- Build Docker image
- Push lên GHCR (GitHub Container Registry — Kho Lưu Trữ Container GitHub)
- Tag image với: `latest`, `git SHA`, `YYYY-MM-DD`
- Chỉ push khi push vào `main` (không push khi PR)

### Đáp Án Mẫu

```yaml
# .github/workflows/docker-build.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
  pull_request:           # Build nhưng không push trên PR

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write     # Cần để push lên GHCR

    steps:
      - uses: actions/checkout@v4

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,prefix=,format=short
            type=raw,value={{date 'YYYY-MM-DD'}}

      - name: Login to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # Không cần secret thêm

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}  # Chỉ push khi không phải PR
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha           # GitHub Actions cache cho Docker layers
          cache-to: type=gha,mode=max
```

---

## Bài Tập 10: Security Scanning Pipeline

**Cấp độ:** ⭐⭐⭐ Trung Bình | **Thời gian:** 25 phút

### Requirements

- Dependency vulnerability scan (Quét Lỗ Hổng Phụ Thuộc) với `npm audit`
- Static analysis (Phân Tích Tĩnh) cơ bản
- Fail workflow nếu có HIGH severity vulnerabilities
- Upload kết quả như SARIF (Static Analysis Results Interchange Format — Định Dạng Kết Quả Phân Tích Tĩnh)

### Đáp Án Mẫu

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 8 * * 1'   # Mỗi thứ Hai lúc 8 giờ sáng

permissions:
  contents: read
  security-events: write  # Cần để upload SARIF

jobs:
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Run npm audit
        run: |
          # Fail nếu có high/critical vulnerabilities
          npm audit --audit-level=high
        continue-on-error: true     # Capture kết quả trước khi quyết định fail

      - name: Generate audit report
        run: npm audit --json > audit-report.json || true

      - uses: actions/upload-artifact@v4
        with:
          name: security-audit-report
          path: audit-report.json

  codeql-analysis:
    name: CodeQL Static Analysis
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript     # Hoặc: python, java, go, cpp

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"
          # Kết quả tự động upload lên Security tab của repo
```

---

## Bài Tập 11: Slack Notifications

**Cấp độ:** ⭐⭐⭐ Trung Bình | **Thời gian:** 20 phút

### Chuẩn Bị

Tạo Slack Webhook URL (Webhook — URL Nhận Thông Báo Tự Động):
- Tạo Slack App → Incoming Webhooks → Activate → Add New Webhook
- Lưu URL vào GitHub Secrets với tên `SLACK_WEBHOOK_URL`

### Requirements

- Gửi notification khi deploy thành công
- Gửi notification khác (màu đỏ) khi deploy thất bại
- Thông báo phải bao gồm: tên app, environment, version, link đến run

### Đáp Án Mẫu

```yaml
# .github/workflows/notify-slack.yml
name: Deploy with Notifications

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        id: deploy
        run: |
          echo "Deploying..."
          # Simulate random success/failure for demo
          exit 0  # Đổi thành exit 1 để test failure notification

      - name: Notify success
        if: success()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "attachments": [{
                "color": "good",
                "title": "✅ Deployment Successful",
                "fields": [
                  {"title": "Repository", "value": "${{ github.repository }}", "short": true},
                  {"title": "Branch", "value": "${{ github.ref_name }}", "short": true},
                  {"title": "Commit", "value": "${{ github.sha }}", "short": true},
                  {"title": "Actor", "value": "${{ github.actor }}", "short": true},
                  {"title": "Workflow", "value": "<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

      - name: Notify failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "attachments": [{
                "color": "danger",
                "title": "❌ Deployment Failed",
                "text": "Deployment failed for *${{ github.repository }}* on branch `${{ github.ref_name }}`.\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Failed Run>"
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## Bài Tập 12: OIDC Authentication với AWS

**Cấp độ:** ⭐⭐⭐⭐ Nâng Cao | **Thời gian:** 45 phút

### Chuẩn Bị Phía AWS

```json
// IAM Trust Policy cho OIDC
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
      },
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      }
    }
  }]
}
```

### Requirements

- Authenticate với AWS sử dụng OIDC (không dùng access keys)
- List S3 buckets để verify credentials hoạt động
- Chỉ allow authentication từ nhánh `main`
- Print ra temporary credential expiration time

### Đáp Án Mẫu

```yaml
# .github/workflows/oidc-aws.yml
name: OIDC AWS Authentication

on:
  push:
    branches: [main]

permissions:
  id-token: write    # Bắt buộc để request OIDC token
  contents: read

jobs:
  aws-auth:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ap-southeast-1
          # role-session-name tùy chọn — giúp audit trail
          role-session-name: GitHubActions-${{ github.run_id }}

      - name: Verify AWS identity
        run: |
          echo "=== AWS Identity ==="
          aws sts get-caller-identity

      - name: List S3 buckets (verify permissions)
        run: |
          echo "=== S3 Buckets ==="
          aws s3 ls

      - name: Show credential expiration
        run: |
          # Temporary credentials expire sau 1 giờ (mặc định)
          echo "Credentials are temporary and expire automatically"
          echo "Run ID for audit: ${{ github.run_id }}"
```

---

## Bài Tập 13: Custom Composite Action

**Cấp độ:** ⭐⭐⭐⭐ Nâng Cao | **Thời gian:** 40 phút

### Requirements

Tạo composite action tên `setup-app` trong thư mục `.github/actions/setup-app/` với:
- Input: `node-version` (default: '20'), `install-command` (default: 'npm ci')
- Output: `cache-hit` (true/false)
- Steps: checkout + setup Node + cache + install
- Sau đó dùng action này trong workflow

### Đáp Án Mẫu

**Action definition — `.github/actions/setup-app/action.yml`:**

```yaml
name: Setup Application
description: Composite action for standard Node.js app setup

inputs:
  node-version:
    description: Node.js version to use
    default: '20'
  install-command:
    description: Command to install dependencies
    default: 'npm ci'

outputs:
  cache-hit:
    description: Whether the cache was hit
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ inputs.node-version }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-${{ inputs.node-version }}-

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: ${{ inputs.install-command }}
      shell: bash

    - name: Verify setup
      run: |
        echo "Node: $(node --version)"
        echo "npm: $(npm --version)"
        echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
      shell: bash
```

**Workflow dùng composite action:**

```yaml
# .github/workflows/use-composite.yml
name: Use Composite Action

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Setup application
        id: setup
        uses: ./.github/actions/setup-app  # Local composite action
        with:
          node-version: '20'
          install-command: 'npm ci'

      - name: Check cache result
        run: echo "Cache was hit: ${{ steps.setup.outputs.cache-hit }}"

      - run: npm test
      - run: npm run build
```

---

## Bài Tập 14: Release Automation

**Cấp độ:** ⭐⭐⭐⭐ Nâng Cao | **Thời gian:** 45 phút

### Requirements

- Trigger khi có git tag với format `v*.*.*` (ví dụ: v1.2.3)
- Build ứng dụng
- Tạo GitHub Release (Phát Hành) tự động với:
  - Release notes từ git commits kể từ lần release trước
  - Build artifact được đính kèm
  - Semantic versioning (Đánh Số Phiên Bản Ngữ Nghĩa) trong release title

### Đáp Án Mẫu

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'   # Chỉ trigger với semantic version tags

permissions:
  contents: write    # Cần để tạo GitHub Release

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # Cần full history để generate changelog

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Create release archive
        run: |
          VERSION=${{ github.ref_name }}
          tar -czf "app-${VERSION}.tar.gz" dist/
          echo "Archive created: app-${VERSION}.tar.gz"

      - name: Generate changelog
        id: changelog
        run: |
          # Lấy tag trước đó
          PREV_TAG=$(git tag --sort=-version:refname | sed -n '2p')
          echo "Previous tag: $PREV_TAG"
          
          if [ -z "$PREV_TAG" ]; then
            CHANGELOG="Initial release"
          else
            # Commits kể từ tag trước
            CHANGELOG=$(git log ${PREV_TAG}..HEAD --pretty=format:"- %s (%h)" --no-merges)
          fi
          
          # Lưu vào file để dùng sau (tránh multiline output issues)
          echo "$CHANGELOG" > CHANGELOG.txt
          cat CHANGELOG.txt

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          name: Release ${{ github.ref_name }}
          body_path: CHANGELOG.txt
          files: |
            app-${{ github.ref_name }}.tar.gz
          draft: false
          prerelease: ${{ contains(github.ref_name, '-rc') || contains(github.ref_name, '-beta') }}
```

**Cách tạo release:**

```bash
# Tạo và push tag
git tag v1.0.0 -m "First release"
git push origin v1.0.0
# → Workflow tự động chạy và tạo GitHub Release
```

---

## Bài Tập 15: Full CI/CD Pipeline End-to-End

**Cấp độ:** ⭐⭐⭐⭐⭐ Thách Thức | **Thời gian:** 2–3 giờ

### Mô Tả

Bài tập tổng hợp — xây dựng pipeline CI/CD hoàn chỉnh từ đầu đến cuối.

### Requirements Đầy Đủ

**CI (khi có PR):**
- [ ] Lint + format check
- [ ] Unit tests với code coverage (>80%)
- [ ] Matrix testing trên Node 18, 20, 22
- [ ] Security vulnerability scan
- [ ] Docker image build (không push)
- [ ] Comment coverage report lên PR

**CD (khi merge vào main):**
- [ ] Build Docker image và push lên GHCR với semantic versioning
- [ ] Deploy lên staging environment tự động
- [ ] Run E2E smoke tests (Kiểm Thử Khói Đầu Cuối)
- [ ] Deploy lên production sau khi có manual approval
- [ ] Slack notification khi deploy thành công/thất bại

**Reusability:**
- [ ] Tách build logic thành composite action
- [ ] Deploy logic thành reusable workflow

**Security:**
- [ ] Pin tất cả actions bởi SHA
- [ ] Minimal permissions cho mỗi job
- [ ] Dùng OIDC cho cloud authentication (nếu có cloud account)

### Cấu Trúc Thư Mục

```
.github/
├── workflows/
│   ├── ci.yml                # CI pipeline cho PRs
│   ├── cd.yml                # CD pipeline cho main
│   ├── reusable-deploy.yml   # Reusable deployment workflow
│   └── release.yml           # Release automation
└── actions/
    ├── build-app/
    │   └── action.yml         # Composite: checkout + setup + build
    └── docker-publish/
        └── action.yml         # Composite: build + tag + push Docker
```

### Checklist Đánh Giá Bản Thân

Sau khi hoàn thành, kiểm tra:

- [ ] Workflow chạy không lỗi trên GitHub
- [ ] Tất cả jobs có tên rõ ràng (không phải "Run step")
- [ ] Mỗi job chỉ có permissions cần thiết
- [ ] Secrets không bao giờ print ra logs
- [ ] Cache hit rate > 70% sau lần chạy đầu tiên
- [ ] Build time CI < 5 phút
- [ ] Có thể rollback bằng cách re-run job cũ

---

## 🎯 Tổng Kết Kỹ Năng

Sau khi hoàn thành tất cả bài tập, bạn đã thực hành:

| Kỹ Năng | Bài Tập |
|---|---|
| Workflow syntax cơ bản | 1, 2 |
| Matrix strategy | 3 |
| Caching | 4 |
| Artifacts | 5 |
| Job outputs | 6 |
| Environments & CD | 7 |
| Reusable workflows | 8 |
| Docker integration | 9 |
| Security scanning | 10 |
| Notifications | 11 |
| OIDC | 12 |
| Custom actions | 13 |
| Release automation | 14 |
| End-to-end pipeline | 15 |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
