# Events & Triggers — Sự Kiện Kích Hoạt Workflow

> Events (Sự kiện) là những gì làm GitHub Actions "thức dậy". Mỗi workflow phải có ít nhất một event. Hiểu đúng events giúp tránh chạy workflow thừa và kiểm soát đúng luồng CI/CD.

## 📋 Mục Lục

1. [Tổng Quan Events](#tổng-quan-events)
2. [Push Events](#push-events)
3. [Pull Request Events](#pull-request-events)
4. [Schedule Events](#schedule-events)
5. [Manual Triggers](#manual-triggers)
6. [Webhook Events](#webhook-events)
7. [Repository Events](#repository-events)
8. [Workflow Events](#workflow-events)
9. [Activity Types](#activity-types)
10. [Filters](#filters)

---

## Tổng Quan Events

GitHub Actions hỗ trợ 3 nhóm events chính:

| Nhóm | Ví Dụ | Dùng Khi |
|---|---|---|
| **Repository Events** | push, pull_request, release | Hoạt động git thông thường |
| **Scheduled Events** | schedule | Chạy định kỳ (cron job) |
| **Manual Events** | workflow_dispatch, workflow_call | Kích hoạt thủ công |

---

## Push Events

### Cú Pháp Cơ Bản

```yaml
on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'          # glob — tất cả nhánh bắt đầu bằng feature/
      - '!feature/wip-*'      # loại trừ nhánh bắt đầu bằng feature/wip-
    branches-ignore:
      - 'dependabot/**'       # bỏ qua PR từ Dependabot
    tags:
      - 'v*'                  # mọi tag bắt đầu bằng v (v1.0, v2.3.1...)
      - 'v[0-9]+.[0-9]+.[0-9]+'  # chỉ semver tag
    tags-ignore:
      - 'v*-beta'             # bỏ qua beta tags
    paths:
      - 'src/**'              # chỉ trigger khi thay đổi trong src/
      - '*.go'
      - 'go.mod'
    paths-ignore:
      - '**.md'               # bỏ qua thay đổi file markdown
      - 'docs/**'
      - '.github/**'
```

### Lưu Ý Quan Trọng

```yaml
# branches: và branches-ignore: KHÔNG dùng chung
# paths: và paths-ignore: KHÔNG dùng chung

# Nếu cả branches và paths được đặt, cả hai phải match mới trigger
on:
  push:
    branches: [main]
    paths: ['src/**']         # chỉ trigger khi push lên main VÀ có thay đổi trong src/
```

### Context Đặc Biệt Cho Push

```yaml
steps:
  - name: Push info
    run: |
      echo "Ref: ${{ github.ref }}"                        # refs/heads/main
      echo "Branch: ${{ github.ref_name }}"               # main
      echo "SHA: ${{ github.sha }}"                        # commit hash
      echo "Before: ${{ github.event.before }}"           # SHA trước push
      echo "Pusher: ${{ github.event.pusher.name }}"      # người push
      echo "Commits: ${{ toJSON(github.event.commits) }}" # danh sách commits
```

---

## Pull Request Events

### Cú Pháp Đầy Đủ

```yaml
on:
  pull_request:
    types:
      - opened          # PR được tạo mới
      - synchronize     # có commit mới push vào branch của PR
      - reopened        # PR bị đóng và mở lại
      - ready_for_review  # PR chuyển từ Draft sang Ready
      - labeled         # label được thêm vào PR
      - unlabeled       # label bị xóa khỏi PR
      - closed          # PR bị đóng (merged hoặc declined)
      - edited          # title/body PR bị chỉnh sửa
    branches:
      - main
      - develop
    paths:
      - 'src/**'
```

**Mặc định** (nếu không khai báo `types:`): `[opened, synchronize, reopened]`

### pull_request vs pull_request_target

| | `pull_request` | `pull_request_target` |
|---|---|---|
| **Context** | Code của PR branch (fork) | Code của base branch |
| **Secrets** | Không có (fork PR) | Có đầy đủ |
| **Quyền write** | Không | Có |
| **Rủi ro** | Thấp | Cao — phải kiểm tra code PR trước khi chạy |
| **Dùng cho** | CI checks thông thường | Auto-label, triage PR từ fork |

```yaml
# ⚠️ CẢNH BÁO BẢO MẬT: pull_request_target chạy với quyền của repo gốc
# Không bao giờ checkout code từ PR trong pull_request_target mà không kiểm tra
on:
  pull_request_target:
    types: [opened, labeled]
```

### Kiểm Tra PR Cụ Thể

```yaml
on:
  pull_request:
    types: [labeled]

jobs:
  deploy-preview:
    # Chỉ deploy khi label 'preview' được thêm
    if: contains(github.event.pull_request.labels.*.name, 'preview')
    runs-on: ubuntu-latest
    steps:
      - run: echo "PR #${{ github.event.pull_request.number }}"
      - run: echo "PR từ: ${{ github.event.pull_request.head.ref }}"
      - run: echo "PR vào: ${{ github.event.pull_request.base.ref }}"
      - run: echo "Author: ${{ github.event.pull_request.user.login }}"
```

---

## Schedule Events

### Cú Pháp Cron

```yaml
on:
  schedule:
    - cron: '0 2 * * *'        # 2:00 AM UTC mỗi ngày
    - cron: '0 9 * * 1-5'      # 9:00 AM UTC thứ Hai–Sáu
    - cron: '*/15 * * * *'     # Mỗi 15 phút (tối thiểu 5 phút)
```

### Định Dạng Cron

```
┌──────────── phút (0–59)
│ ┌────────── giờ (0–23)
│ │ ┌──────── ngày trong tháng (1–31)
│ │ │ ┌────── tháng (1–12)
│ │ │ │ ┌──── ngày trong tuần (0–6, 0=Chủ Nhật)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

| Ký Hiệu | Ý Nghĩa | Ví Dụ |
|---|---|---|
| `*` | Mọi giá trị | `* * * * *` — mỗi phút |
| `,` | Danh sách | `1,3,5` — phút 1, 3, và 5 |
| `-` | Khoảng | `1-5` — từ 1 đến 5 |
| `/` | Bước | `*/15` — mỗi 15 đơn vị |

### Ví Dụ Thực Tế

```yaml
on:
  schedule:
    # Chạy integration tests lúc 3 AM UTC mỗi ngày
    - cron: '0 3 * * *'

    # Backup database vào 0:00 UTC mỗi Chủ Nhật
    - cron: '0 0 * * 0'

    # Kiểm tra expired certs mỗi thứ Hai đầu tháng
    - cron: '0 8 1-7 * 1'

    # Cleanup old artifacts mỗi thứ Sáu 5 PM UTC
    - cron: '0 17 * * 5'
```

### Lưu Ý Schedule

- Schedule workflows chỉ chạy trên nhánh **mặc định** (default branch)
- Thời gian là **UTC** — cần chuyển đổi sang timezone của bạn
- GitHub có thể delay schedule nếu server tải cao (không đảm bảo chính xác đến giây)
- Nếu không có activity trong 60 ngày, GitHub tự **vô hiệu hóa** scheduled workflows

---

## Manual Triggers

### workflow_dispatch — Kích Hoạt Thủ Công

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Môi trường deploy'
        required: true
        type: choice
        options:
          - staging
          - production
        default: staging

      version:
        description: 'Phiên bản deploy (vd: 1.2.3)'
        required: false
        type: string

      dry-run:
        description: 'Chỉ mô phỏng, không thực sự deploy'
        required: false
        type: boolean
        default: false

      log-level:
        description: 'Mức độ log'
        required: false
        type: choice
        options:
          - info
          - debug
          - warning

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy info
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Dry run: ${{ inputs.dry-run }}"
```

**Cách kích hoạt:**
1. GitHub UI: Actions tab → chọn workflow → "Run workflow"
2. GitHub CLI: `gh workflow run deploy.yml -f environment=staging`
3. GitHub API: POST `/repos/{owner}/{repo}/actions/workflows/{workflow_id}/dispatches`

### repository_dispatch — Kích Hoạt Từ Bên Ngoài

```yaml
on:
  repository_dispatch:
    types:
      - deploy-app             # custom event types
      - rollback-deploy

jobs:
  handle-dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Event payload
        run: |
          echo "Event type: ${{ github.event.action }}"
          echo "Payload: ${{ toJSON(github.event.client_payload) }}"
```

**Kích hoạt qua API:**

```bash
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/owner/repo/dispatches \
  -d '{
    "event_type": "deploy-app",
    "client_payload": {
      "version": "1.2.3",
      "environment": "staging"
    }
  }'
```

---

## Webhook Events

### release — Sự Kiện Release

```yaml
on:
  release:
    types:
      - published     # Release được publish (đã release chính thức)
      - created       # Release draft được tạo
      - prereleased   # Pre-release được publish
      - released      # Cả released và prereleased

jobs:
  publish:
    if: github.event.action == 'published' && !github.event.release.prerelease
    runs-on: ubuntu-latest
    steps:
      - run: echo "Releasing ${{ github.event.release.tag_name }}"
      - run: echo "Body: ${{ github.event.release.body }}"
```

### issues — Sự Kiện Issue

```yaml
on:
  issues:
    types: [opened, labeled, closed]

jobs:
  triage:
    if: github.event.action == 'opened'
    runs-on: ubuntu-latest
    steps:
      - name: Auto-label new issues
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              labels: ['needs-triage']
            })
```

### issue_comment — Bình Luận Issue / PR

```yaml
on:
  issue_comment:
    types: [created]

jobs:
  slash-command:
    # Chỉ xử lý comment "/deploy staging" trên PR
    if: |
      github.event.issue.pull_request &&
      startsWith(github.event.comment.body, '/deploy')
    runs-on: ubuntu-latest
    steps:
      - run: |
          COMMAND="${{ github.event.comment.body }}"
          echo "Command: $COMMAND"
```

---

## Repository Events

```yaml
on:
  # Khi file trong repo thay đổi (qua UI hoặc API)
  create:          # nhánh hoặc tag được tạo
  delete:          # nhánh hoặc tag bị xóa
  fork:            # repo được fork
  star:            # repo được star/unstar
    types: [created, deleted]
  watch:           # ai đó bắt đầu watch repo
  page_build:      # GitHub Pages được build
  gollum:          # Wiki page được cập nhật
  check_run:       # check run hoàn thành
  check_suite:     # check suite hoàn thành
  status:          # commit status thay đổi
  deployment:      # deployment được tạo
  deployment_status: # deployment status thay đổi
```

---

## Workflow Events

### workflow_call — Reusable Workflow (Workflow Tái Sử Dụng)

```yaml
# Workflow được gọi (called workflow)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string          # string, boolean, number
      debug:
        required: false
        type: boolean
        default: false
    secrets:
      DEPLOY_KEY:
        required: true
      OPTIONAL_SECRET:
        required: false
    outputs:
      deployment-url:
        description: 'URL sau khi deploy'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - name: Deploy
        id: deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

```yaml
# Workflow gọi (caller workflow)
jobs:
  call-deploy:
    uses: my-org/shared-workflows/.github/workflows/deploy.yml@main
    with:
      environment: staging
      debug: false
    secrets:
      DEPLOY_KEY: ${{ secrets.STAGING_DEPLOY_KEY }}

  use-output:
    needs: call-deploy
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deployed to ${{ needs.call-deploy.outputs.deployment-url }}"
```

---

## Activity Types

Nhiều events có **activity types** để lọc chính xác hơn:

```yaml
on:
  pull_request:
    types: [opened, synchronize]    # chỉ hai trường hợp này

  issues:
    types: [opened, labeled, closed]

  release:
    types: [published]

  label:
    types: [created, deleted, edited]

  milestone:
    types: [opened, closed]

  project:
    types: [created, deleted]

  deployment:
    # không có types — chỉ một activity

  check_run:
    types: [completed, rerequested]
```

---

## Filters — Bộ Lọc

### Glob Patterns

```yaml
branches:
  - main              # chính xác
  - 'release/*'       # release/v1, release/2.0, release/hotfix
  - 'feature/**'      # feature/my-feat, feature/team/my-feat (đa cấp)
  - 'v[0-9]+'         # v1, v2, v10 (regex-like)
  - '!docs/**'        # loại trừ (dùng ! đứng đầu)
```

### Kết Hợp branches và paths

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'api/**'
      - 'shared/**'
    paths-ignore:
      - 'api/docs/**'
      - '**.test.ts'
```

**Logic:** `branches` AND (`paths` OR NOT `paths-ignore`)
- Cả hai điều kiện phải thỏa mãn mới trigger
- `paths` và `paths-ignore` không dùng chung

---

## So Sánh Events — Khi Nào Dùng Cái Nào

| Tình Huống | Event Phù Hợp |
|---|---|
| CI chạy khi merge code | `push` vào main |
| CI chạy khi mở/update PR | `pull_request` [opened, synchronize] |
| Deploy staging từ PR | `pull_request` [labeled] (label = deploy) |
| Deploy production sau release | `release` [published] |
| Cleanup lúc nửa đêm | `schedule` cron |
| Deploy thủ công với params | `workflow_dispatch` |
| Kích hoạt từ system khác | `repository_dispatch` |
| Tái sử dụng workflow | `workflow_call` |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa `push` trigger và `pull_request` trigger?**

A:
- `push`: Trigger khi code được push lên repository. Chạy với quyền đầy đủ của repo (có secrets).
- `pull_request`: Trigger khi có hoạt động trên PR. Với PRs từ fork, **không có secrets** của repo gốc vì lý do bảo mật. Code được checkout là code của PR branch.

Thực tế: CI checks thường dùng `pull_request`; deployment thường dùng `push` vào nhánh protected hoặc `release`.

**Q: `workflow_dispatch` khác `repository_dispatch` như thế nào?**

A:
- `workflow_dispatch`: Kích hoạt thủ công qua GitHub UI, CLI, hoặc API. Có thể định nghĩa inputs. Chỉ người có quyền write repo mới chạy được.
- `repository_dispatch`: Kích hoạt qua GitHub API với custom event type và payload tùy ý. Dùng cho tích hợp với hệ thống bên ngoài (webhook từ CD system khác, monitoring alerts...).

**Q: Tại sao schedule workflow không chạy trên feature branches?**

A: GitHub Actions chỉ chạy schedule workflows trên **default branch** (thường là main). Nếu muốn test schedule trên branch khác, phải tạm thời merge vào main hoặc dùng `workflow_dispatch` để simulate.

---

## 📂 Điều Hướng

- [← Workflow Syntax](1-workflow-syntax.md)
- [→ Runners](3-runners.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
