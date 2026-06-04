# Concurrency — Kiểm Soát Đồng Thời

> Concurrency Control (Kiểm Soát Đồng Thời) giải quyết vấn đề nhiều workflow runs chạy cùng lúc: ngăn deploy chồng chéo, race conditions (điều kiện tranh chấp), và lãng phí tài nguyên.

---

## 📚 Mục Lục

1. [Vấn Đề Không Có Concurrency Control](#1-vấn-đề-không-có-concurrency-control)
2. [Cú Pháp Cơ Bản](#2-cú-pháp-cơ-bản)
3. [Concurrency Groups — Nhóm Đồng Thời](#3-concurrency-groups--nhóm-đồng-thời)
4. [cancel-in-progress — Hủy Run Đang Chạy](#4-cancel-in-progress--hủy-run-đang-chạy)
5. [Concurrency ở Cấp Job](#5-concurrency-ở-cấp-job)
6. [Patterns Thực Tế](#6-patterns-thực-tế)
7. [Concurrency với Environments](#7-concurrency-với-environments)
8. [Giới Hạn Và Hành Vi Đặc Biệt](#8-giới-hạn-và-hành-vi-đặc-biệt)
9. [Anti-Patterns](#9-anti-patterns)
10. [Tóm Tắt](#10-tóm-tắt)

---

## 1. Vấn Đề Không Có Concurrency Control

### Scenario (Kịch Bản) Thực Tế

```
Developer A push lúc 10:00 → Run #1 bắt đầu deploy
Developer B push lúc 10:01 → Run #2 bắt đầu deploy
Developer C push lúc 10:02 → Run #3 bắt đầu deploy

Run #1 deploy phiên bản A
Run #2 deploy phiên bản B (chèn lên A)
Run #3 deploy phiên bản C (chèn lên B)

Kết quả: Production chạy phiên bản C nhưng:
- Migrations từ A và B đã chạy không theo thứ tự
- Load balancer có thể đang chỉ vào instances khác nhau
- Race condition → hệ thống không nhất quán
```

### Tình Huống Phổ Biến Cần Concurrency Control

| Tình Huống | Rủi Ro Nếu Không Kiểm Soát |
|---|---|
| Deploy lên production | Race condition, trạng thái không nhất quán |
| Database migration | Migration chạy hai lần hoặc sai thứ tự |
| PR preview environments | Nhiều deploy cùng PR xung đột |
| Build artifacts | File bị ghi đè bởi build song song |
| Integration tests | Test dùng chung DB bị nhiễu |

---

## 2. Cú Pháp Cơ Bản

### Cú Pháp Ở Cấp Workflow

```yaml
name: Deploy

on:
  push:
    branches: [main]

# Concurrency ở cấp workflow — áp dụng cho toàn bộ workflow
concurrency:
  group: deploy-production
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Các Trường Trong concurrency

```yaml
concurrency:
  # group — tên nhóm, các runs cùng group thì serialized (tuần tự hóa)
  group: <tên-nhóm>

  # cancel-in-progress — có hủy run đang chạy khi có run mới không
  # true  → hủy run cũ, ưu tiên run mới
  # false → run mới phải chờ run cũ hoàn thành (queue — hàng đợi)
  cancel-in-progress: true | false
```

---

## 3. Concurrency Groups — Nhóm Đồng Thời

### Tên Group Tĩnh (Static Group Name)

```yaml
# Tất cả pushes lên main dùng chung một group
concurrency:
  group: deploy-main
  cancel-in-progress: true
```

**Cách hoạt động:**
```
Push #1 → group "deploy-main" → Run #1 bắt đầu
Push #2 → group "deploy-main" → Nếu cancel-in-progress=true: hủy Run #1, bắt đầu Run #2
                              → Nếu cancel-in-progress=false: Run #2 đợi Run #1 xong
```

### Tên Group Động (Dynamic Group Name)

```yaml
# Mỗi branch có group riêng — không ảnh hưởng lẫn nhau
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

**Ví dụ:**
```
feature/login  → group: "CI-refs/heads/feature/login"
feature/signup → group: "CI-refs/heads/feature/signup"
main           → group: "CI-refs/heads/main"
# → 3 nhóm độc lập, không chặn nhau
```

### Các Expression Phổ Biến Cho Group Name

```yaml
# Theo workflow + branch
group: ${{ github.workflow }}-${{ github.ref }}

# Theo workflow + PR number
group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}

# Theo workflow + SHA (mỗi commit là một nhóm riêng — hiếm dùng)
group: ${{ github.workflow }}-${{ github.sha }}

# Theo environment + branch
group: deploy-${{ inputs.environment }}-${{ github.ref_name }}

# Cố định cho toàn bộ production
group: production-deploy
```

---

## 4. cancel-in-progress — Hủy Run Đang Chạy

### `cancel-in-progress: true`

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

```
Timeline:
t=0  → Push #1 → Run A bắt đầu (testing...)
t=30 → Push #2 → Run B trigger → Run A bị HỦY → Run B bắt đầu
t=60 → Push #3 → Run C trigger → Run B bị HỦY → Run C bắt đầu

Kết quả: Chỉ Run C chạy đến hết. A và B bị cancel.
Minutes tiêu thụ: ~30s (A) + ~30s (B) + full (C) thay vì full×3
```

**Phù hợp với:** PR checks, CI tests — muốn kết quả của code mới nhất.

### `cancel-in-progress: false`

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

```
Timeline:
t=0  → Push #1 → Run A bắt đầu (deploying...)
t=30 → Push #2 → Run B trigger → Run B CHỜ (queued)
t=120→ Run A hoàn thành → Run B bắt đầu
t=240→ Run B hoàn thành

Kết quả: Cả hai deploy thành công, tuần tự.
```

**Phù hợp với:** Production deploys, database migrations — không được bỏ qua.

### Chọn Giá Trị Phù Hợp

| Trường Hợp | cancel-in-progress |
|---|---|
| PR/feature branch CI | `true` — muốn kết quả code mới nhất |
| Preview environments | `true` — PR mới cần preview mới |
| Main branch CI | `true` hoặc `false` tùy policy |
| Staging deploy | `false` — đảm bảo mọi commit được test |
| Production deploy | `false` — không được bỏ qua |
| Database migrations | `false` — thứ tự rất quan trọng |

---

## 5. Concurrency ở Cấp Job

### Tại Sao Cần Concurrency Cấp Job?

Concurrency cấp workflow áp dụng cho toàn bộ workflow. Nhưng đôi khi bạn muốn:
- Jobs `test` chạy song song không giới hạn
- Chỉ job `deploy` mới bị giới hạn

```yaml
name: CI/CD Pipeline

on: push

jobs:
  # Job test: không có concurrency limit — chạy thoải mái
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  # Job deploy: có concurrency riêng — chỉ 1 deploy cùng lúc
  deploy:
    needs: test
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-${{ github.ref_name }}
      cancel-in-progress: false
    steps:
      - run: ./deploy.sh
```

### So Sánh Cấp Workflow vs Cấp Job

```yaml
# Cấp Workflow: hủy/chặn từ đầu, kể cả test chưa chạy
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test: ...    # bị hủy luôn
  deploy: ...  # bị hủy luôn
```

```yaml
# Cấp Job: test chạy bình thường, chỉ deploy mới bị kiểm soát
jobs:
  test:
    runs-on: ubuntu-latest
    # Không có concurrency → chạy song song tự do
    steps: ...

  deploy:
    needs: test
    concurrency:
      group: deploy-production
      cancel-in-progress: false
    steps: ...
```

---

## 6. Patterns Thực Tế

### Pattern 1: CI Cho Pull Requests

```yaml
name: CI

on:
  pull_request:
    branches: [main]

# Mỗi PR có group riêng, push mới hủy CI cũ của cùng PR
concurrency:
  group: ci-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint
```

```
PR #42, Push A → group "ci-pr-42" → test + lint bắt đầu
PR #42, Push B → group "ci-pr-42" → test + lint bị HỦY → bắt đầu mới
PR #43, Push C → group "ci-pr-43" → Nhóm khác, không ảnh hưởng PR #42
```

### Pattern 2: Deploy Pipeline Với Queue

```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-staging
      cancel-in-progress: true   # Staging: ưu tiên code mới nhất
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: deploy-production
      cancel-in-progress: false  # Production: KHÔNG bỏ qua bất kỳ deploy nào
    steps:
      - run: ./deploy.sh production
```

### Pattern 3: Preview Environments Cho PRs

```yaml
name: PR Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    concurrency:
      # Mỗi PR có preview environment riêng
      group: preview-pr-${{ github.event.pull_request.number }}
      cancel-in-progress: true   # Push mới = preview mới ngay lập tức
    steps:
      - uses: actions/checkout@v4
      - name: Deploy preview
        run: |
          ./deploy-preview.sh pr-${{ github.event.pull_request.number }}
      - name: Comment PR with URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed: https://pr-${{ github.event.pull_request.number }}.preview.example.com'
            })

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Remove preview environment
        run: ./cleanup-preview.sh pr-${{ github.event.pull_request.number }}
```

### Pattern 4: Scheduled Jobs Không Bị Chồng Chéo

```yaml
name: Nightly Data Sync

on:
  schedule:
    - cron: '0 2 * * *'   # Mỗi đêm lúc 2h sáng
  workflow_dispatch:        # Cho phép trigger thủ công

concurrency:
  group: nightly-sync
  # false: nếu sync đêm qua chưa xong và trigger lại → đợi
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    timeout-minutes: 180   # 3 giờ timeout
    steps:
      - run: ./sync-all-data.sh
```

---

## 7. Concurrency với Environments

### GitHub Environments Có Concurrency Riêng

GitHub Environments (Môi Trường GitHub) có cơ chế protection rules riêng, hoạt động song song với `concurrency` block:

```yaml
jobs:
  deploy:
    environment: production
    # environment.protection_rules → yêu cầu approval trước khi chạy
    # concurrency → kiểm soát bao nhiêu deploy chạy cùng lúc
    concurrency:
      group: production-deploy
      cancel-in-progress: false
```

```
Flow:
1. Job trigger → Chờ Environment approval (nếu có required reviewers)
2. Sau khi approved → Vào concurrency group
3. Nếu có deploy khác đang chạy → Chờ (cancel-in-progress: false)
4. Deploy chạy
```

### Multi-Environment Pipeline

```yaml
name: Multi-Stage Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: development
    concurrency:
      group: deploy-dev
      cancel-in-progress: true   # dev: luôn deploy code mới nhất

  deploy-staging:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging
    concurrency:
      group: deploy-staging
      cancel-in-progress: false  # staging: đợi turn

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production      # production cần approval
    concurrency:
      group: deploy-prod
      cancel-in-progress: false  # production: tuyệt đối không cancel
```

---

## 8. Giới Hạn Và Hành Vi Đặc Biệt

### Queue Behavior (Hành Vi Hàng Đợi)

Với `cancel-in-progress: false`:
- Chỉ có **một run đang pending** (chờ) được phép tồn tại
- Run thứ ba sẽ thay thế run đang pending (không phải run đang chạy)

```
Run A đang chạy
Run B đang pending (chờ A xong)
Run C trigger → Run C thay thế Run B trong hàng đợi
→ Khi A xong, C chạy (không phải B)
```

### Scope (Phạm Vi) Của Concurrency Group

Concurrency groups có phạm vi trong một **repository** — không chia sẻ giữa repositories khác nhau, ngay cả khi cùng organization.

### Cleanup Khi Cancel

Khi một run bị cancel do concurrency:
- Các steps đang chạy được gửi signal SIGTERM
- Có 5 giây để cleanup
- Jobs dùng `post` steps (cleanup) vẫn chạy

```yaml
steps:
  - name: Deploy
    id: deploy
    run: ./deploy.sh

  # Step này chạy kể cả khi bị cancel
  - name: Rollback on cancel
    if: cancelled() && steps.deploy.outcome == 'success'
    run: ./rollback.sh
```

---

## 9. Anti-Patterns

### ❌ Group Name Quá Rộng

```yaml
# Sai: tất cả workflows trong repo dùng chung một group
# → CI feature/A blocks CI feature/B
concurrency:
  group: ci-all-workflows
  cancel-in-progress: true
```

```yaml
# Đúng: phân biệt theo workflow + branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### ❌ cancel-in-progress: true Cho Production

```yaml
# Sai: Nếu push #2 cancel push #1 đang deploy production
# → Production chỉ deploy một nửa, bị hủy giữa chừng → sự cố!
jobs:
  deploy-prod:
    concurrency:
      group: deploy-production
      cancel-in-progress: true   # NGUY HIỂM cho production!
```

```yaml
# Đúng: Production phải queue, không bao giờ cancel
jobs:
  deploy-prod:
    concurrency:
      group: deploy-production
      cancel-in-progress: false
```

### ❌ Không Có Concurrency Cho Scheduled Jobs

```yaml
# Sai: Nếu job chạy hơn 1h mà cron là mỗi giờ
# → hai runs chạy song song, conflict dữ liệu
on:
  schedule:
    - cron: '0 * * * *'

# Không có concurrency → hai runs có thể chạy đồng thời
```

```yaml
# Đúng: Đảm bảo chỉ 1 scheduled run tại một thời điểm
concurrency:
  group: scheduled-job
  cancel-in-progress: false
```

---

## 10. Tóm Tắt

### Cheat Sheet

```yaml
# Template cho CI (PR checks)
concurrency:
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

# Template cho Staging deploy
concurrency:
  group: deploy-staging-${{ github.ref_name }}
  cancel-in-progress: true   # Staging: code mới nhất là ưu tiên

# Template cho Production deploy
concurrency:
  group: deploy-production
  cancel-in-progress: false  # Production: KHÔNG bao giờ hủy

# Template cho Scheduled jobs
concurrency:
  group: scheduled-${{ github.workflow }}
  cancel-in-progress: false  # Đảm bảo không chồng chéo
```

### Câu Hỏi Phỏng Vấn

**Q: `concurrency` trong GitHub Actions giải quyết vấn đề gì?**

> Ngăn nhiều runs cùng nhóm chạy đồng thời, tránh race conditions và xung đột tài nguyên — đặc biệt quan trọng khi deploy. Với `cancel-in-progress: true` tiết kiệm minutes bằng cách hủy runs cũ; với `false` đảm bảo mọi deploy đều được thực hiện theo thứ tự.

**Q: Sự khác nhau giữa `cancel-in-progress: true` và `false`?**

> `true`: run mới hủy run đang chạy — phù hợp CI/PR checks. `false`: run mới đợi trong hàng đợi — bắt buộc cho production deploy. Lưu ý: với `false`, chỉ giữ một pending run, run thứ ba sẽ thay thế pending run thứ hai.

**Q: Concurrency group scope là gì?**

> Phạm vi trong một repository. Các repositories khác nhau không chia sẻ concurrency groups, ngay cả cùng organization.

---

**Cập Nhật:** 2026-05-11 | **Tiếp Theo:** [3-fan-out-fan-in.md](3-fan-out-fan-in.md)
