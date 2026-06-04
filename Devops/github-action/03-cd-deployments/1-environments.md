# 1 — GitHub Environments (Môi Trường Triển Khai)

> GitHub Environments cho phép bạn định nghĩa các môi trường deployment (triển khai) như `staging`, `production` với quy tắc bảo vệ, secrets riêng biệt và lịch sử triển khai có thể kiểm tra.

## 🎯 Environments là gì?

Environments là một tính năng của GitHub cho phép bạn mô hình hóa các môi trường thực tế của ứng dụng:

```
Code Repository
  └── Environments
        ├── development   (tự động, không cần approval)
        ├── staging       (tự động sau CI pass)
        └── production    (cần approval từ senior engineer)
```

Mỗi environment có thể có:
- **Protection Rules** (Quy Tắc Bảo Vệ) — ai có thể approve, phải đợi bao lâu
- **Secrets** (Bí Mật) — thông tin nhạy cảm riêng cho từng môi trường
- **Variables** (Biến) — cấu hình riêng cho từng môi trường
- **Deployment history** (Lịch Sử Triển Khai) — biết ai deploy gì lúc nào

---

## ⚙️ Tạo và Cấu Hình Environment

### Tạo Environment qua GitHub UI

```
Repository → Settings → Environments → New environment

Điền tên: production
```

### Protection Rules (Quy Tắc Bảo Vệ)

#### Required Reviewers (Người Phê Duyệt Bắt Buộc)

```
✅ Required reviewers: @alice, @bob, team:backend-leads
   Tối đa 6 reviewers
   Chỉ cần 1 người approve là đủ (hoặc cấu hình số lượng)
```

Khi workflow chạy đến job có `environment: production`, GitHub sẽ:
1. Dừng workflow và gửi notification cho reviewers
2. Reviewers vào GitHub UI để Approve hoặc Reject
3. Nếu Approve → workflow tiếp tục
4. Nếu Reject → workflow bị cancelled

#### Wait Timer (Thời Gian Chờ)

```
✅ Wait timer: 10 minutes

Dùng để:
- Có thời gian cancel nếu phát hiện lỗi sau CI
- Buffer time (khoảng đệm) cho team chuẩn bị
- Tránh deploy quá nhanh vào giờ cao điểm
```

#### Deployment Branches (Nhánh Được Phép Deploy)

```
Options:
  - All branches    — mọi nhánh đều có thể deploy (không khuyến nghị cho production)
  - Protected branches only — chỉ nhánh được bảo vệ (thường là main)
  - Selected branches — chỉ nhánh theo pattern (ví dụ: release/*)
```

```yaml
# Ví dụ: Chỉ cho phép deploy từ nhánh main và release/*
Deployment branches: Selected branches
  Patterns:
    - main
    - release/*
    - hotfix/*
```

#### Prevent Self-Review (Ngăn Tự Phê Duyệt)

```
✅ Prevent self-review: enabled

Người tạo PR/push code không thể tự approve deployment của họ.
Quan trọng cho compliance (tuân thủ) và kiểm soát nội bộ.
```

---

## 🔑 Environment Secrets và Variables

### Secrets — Độ Ưu Tiên Khi Tra Cứu

```
Khi workflow chạy trong environment "production":

Độ ưu tiên (cao → thấp):
  1. Environment Secret (production.DATABASE_URL)
  2. Repository Secret (DATABASE_URL)
  3. Organization Secret (DATABASE_URL)

Environment secret ghi đè lên repo/org secret cùng tên!
```

### Thêm Environment Secret qua UI

```
Repository → Settings → Environments → production
  → Add secret
  → Name: DATABASE_URL
  → Value: postgres://prod-host:5432/mydb
```

### Sử Dụng Environment Secret trong Workflow

```yaml
jobs:
  deploy-production:
    environment: production           # Kích hoạt environment secrets
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          # ${{ secrets.DATABASE_URL }} lấy từ production environment
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: ./scripts/deploy.sh
```

### Environment Variables (Biến Môi Trường)

```yaml
# Trên GitHub UI:
# Settings → Environments → staging → Add variable
# Name: APP_URL, Value: https://staging.myapp.com

jobs:
  deploy:
    environment: staging
    steps:
      - run: echo "Deploying to ${{ vars.APP_URL }}"
```

### Cấu Hình Điển Hình cho Nhiều Môi Trường

```
Environment: development
  Variables:
    APP_URL=https://dev.myapp.com
    LOG_LEVEL=debug
    REPLICAS=1
  Secrets:
    DATABASE_URL=postgres://dev-db/myapp_dev

Environment: staging
  Variables:
    APP_URL=https://staging.myapp.com
    LOG_LEVEL=info
    REPLICAS=2
  Secrets:
    DATABASE_URL=postgres://staging-db/myapp_staging

Environment: production
  Protection Rules:
    Required reviewers: @lead-engineer
    Wait timer: 5 minutes
    Deployment branches: main
  Variables:
    APP_URL=https://myapp.com
    LOG_LEVEL=warn
    REPLICAS=5
  Secrets:
    DATABASE_URL=postgres://prod-db/myapp_prod
```

---

## 🔄 Deployment Flow Đầy Đủ

### Multi-Environment Pipeline

```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main]

jobs:
  # ── Staging: Tự động, không cần approval ──────────────
  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: ${{ vars.APP_URL }}    # Hiển thị link trong GitHub UI

    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: ./deploy.sh staging

  # ── Smoke tests sau khi deploy staging ──────────────
  smoke-tests:
    name: Smoke Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Test staging
        run: |
          curl -f https://staging.myapp.com/health || exit 1
          curl -f https://staging.myapp.com/api/version || exit 1

  # ── Production: Cần approval ────────────────────────
  deploy-production:
    name: Deploy → Production
    needs: smoke-tests
    runs-on: ubuntu-latest
    environment:
      name: production             # GitHub sẽ yêu cầu approval ở đây
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: ./deploy.sh production

      - name: Post-deploy verification
        run: |
          sleep 30
          curl -f https://myapp.com/health
```

### Xử Lý Khi Deployment Bị Reject

```yaml
  deploy-production:
    environment: production

    steps:
      - name: Deploy
        id: deploy
        run: ./deploy.sh

      - name: Notify team on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text": "✅ Production deployment successful: ${{ github.sha }}"}'
          webhook: ${{ secrets.SLACK_WEBHOOK }}

      # Nếu reviewer reject, job bị cancelled → bước này không chạy
      # Nếu deploy thất bại sau khi approve:
      - name: Alert on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text": "❌ Production deployment FAILED: ${{ github.sha }} — @oncall please check!"}'
          webhook: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 📊 Deployment Status và Tracking

### Deployment Status trong GitHub UI

Khi dùng `environment` trong job, GitHub tự động:
- Tạo deployment record (bản ghi triển khai) cho commit đó
- Hiển thị status: pending → in_progress → success / failure
- Link đến environment URL
- Lưu lịch sử — ai approve, khi nào, từ commit nào

```
Repository → Deployments (bên phải trang chủ repo)
  ├── production (active) — v1.2.3 — deployed 2 hours ago by @alice
  ├── staging (active) — v1.2.4 — deployed 10 minutes ago
  └── production (inactive) — v1.2.2 — 2 days ago
```

### Deployment Protection Rules API

```yaml
# Dùng GitHub REST API để tự động tạo deployment
- name: Create deployment
  uses: actions/github-script@v7
  with:
    script: |
      const deployment = await github.rest.repos.createDeployment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        ref: context.sha,
        environment: 'production',
        required_contexts: [],   // Bỏ qua status check requirement
        description: 'Deploying version ${{ github.sha }}'
      });

      await github.rest.repos.createDeploymentStatus({
        owner: context.repo.owner,
        repo: context.repo.repo,
        deployment_id: deployment.data.id,
        state: 'success',
        environment_url: 'https://myapp.com',
        description: 'Deployment complete'
      });
```

---

## 🔧 Patterns Thực Tế

### Pattern 1: One-Click Production Deploy

```yaml
# Cho phép trigger thủ công deployment lên bất kỳ env nào
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to which environment?'
        required: true
        type: environment    # Dropdown liệt kê tất cả environments

jobs:
  deploy:
    environment: ${{ inputs.environment }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
```

### Pattern 2: Hotfix Fast-Track (Triển Khai Khẩn Cấp)

```yaml
on:
  push:
    branches: [hotfix/*]

jobs:
  deploy-hotfix:
    environment: production-hotfix    # Environment riêng, ít reviewers hơn
    # ...
```

### Pattern 3: Rollback Job

```yaml
jobs:
  rollback:
    name: Rollback Production
    environment: production          # Vẫn cần approval để rollback
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch'

    steps:
      - name: Get previous stable version
        id: prev
        run: |
          PREV=$(git tag --sort=-creatordate | sed -n '2p')
          echo "version=$PREV" >> $GITHUB_OUTPUT

      - name: Deploy previous version
        run: ./deploy.sh ${{ steps.prev.outputs.version }}
```

### Pattern 4: Environment-Specific Configuration

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging, production]
    environment:
      name: ${{ matrix.environment }}

    steps:
      - name: Deploy
        env:
          APP_URL: ${{ vars.APP_URL }}             # Từ environment variable
          DB_URL: ${{ secrets.DATABASE_URL }}      # Từ environment secret
        run: |
          echo "Deploying to: ${{ matrix.environment }}"
          echo "URL: $APP_URL"
          ./deploy.sh
```

---

## ⚡ Concurrency Control (Kiểm Soát Đồng Thời) cho Deployments

```yaml
jobs:
  deploy-production:
    # Đảm bảo chỉ có 1 deployment production chạy tại một thời điểm
    concurrency:
      group: production-deployment
      cancel-in-progress: false    # KHÔNG cancel đang chạy — đợi hoàn thành
    environment: production
    # ...
```

**Lý do `cancel-in-progress: false` cho production:**
- Deployment đang chạy có thể đang ở trạng thái giữa chừng
- Cancel giữa chừng có thể gây ra trạng thái không nhất quán
- An toàn hơn khi đợi deployment hiện tại hoàn thành rồi mới chạy cái mới

---

## ❌ Những Lỗi Phổ Biến

### 1. Quên khai báo `environment` — mất protection rules

```yaml
# ❌ SAI — không có protection rules, không có environment secrets
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production

# ✅ ĐÚNG
jobs:
  deploy-production:
    environment: production    # Khai báo environment ở đây
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### 2. Dùng repo secrets thay vì environment secrets

```yaml
# ❌ SAI — dùng chung 1 secret cho tất cả environments
env:
  DB_URL: ${{ secrets.DATABASE_URL }}    # Luôn dùng production DB!

# ✅ ĐÚNG — mỗi environment có secret riêng
# staging environment → secrets.DATABASE_URL = staging DB URL
# production environment → secrets.DATABASE_URL = prod DB URL
jobs:
  deploy:
    environment: ${{ inputs.environment }}
    env:
      DB_URL: ${{ secrets.DATABASE_URL }}    # Lấy từ đúng environment
```

### 3. Thiếu `url` trong environment

```yaml
# ❌ Mất link trong GitHub Deployments UI
environment:
  name: production

# ✅ Có link tiện lợi để verify sau khi deploy
environment:
  name: production
  url: https://myapp.com    # Hoặc dùng output từ deploy step
```

---

## 📋 Checklist Environments

Trước khi đưa vào production, kiểm tra:

- [ ] Environment `staging` và `production` đã được tạo
- [ ] Production có Required Reviewers (tối thiểu 1 senior)
- [ ] Production có Deployment Branches: chỉ `main`
- [ ] Environment secrets tách biệt (không dùng chung với repo secrets)
- [ ] `environment: name + url` được khai báo trong tất cả deployment jobs
- [ ] Concurrency group được đặt cho production deployments
- [ ] Smoke tests chạy sau khi deploy staging, trước khi deploy production
- [ ] Có notification step khi deployment thành công / thất bại

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [README.md](README.md) | 1-environments.md | [2-deployment-strategies.md](2-deployment-strategies.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
