# Branch Protection — Bảo Vệ Nhánh Và Kiểm Soát Merge

> Branch protection rules (Quy Tắc Bảo Vệ Nhánh) là lớp kiểm soát cuối cùng trước khi code vào `main`. Status checks — required reviews — merge restrictions cộng lại tạo ra một "quality gate" mà không ai có thể bỏ qua, kể cả admin.

## 📋 Mục Lục

1. [Status Checks](#status-checks)
2. [Branch Protection Rules](#branch-protection-rules)
3. [Required Reviews](#required-reviews)
4. [Merge Strategies](#merge-strategies)
5. [CODEOWNERS](#codeowners)
6. [Rulesets — GitHub Enterprise](#rulesets)
7. [Merge Queue](#merge-queue)

---

## Status Checks

Status Check (Kiểm Tra Trạng Thái) — kết quả pass/fail từ CI workflow được ghi vào commit. GitHub dùng để quyết định có cho merge không.

### Cách Status Checks Hoạt Động

```
Developer mở Pull Request
          │
          ▼
GitHub kích hoạt CI workflows
          │
    ┌─────┴──────┐
    ▼            ▼
 CI workflow   CI workflow
 "lint"        "test"
    │              │
    ▼              ▼
 ✅ Pass       ✅ Pass
    │              │
    └──────┬───────┘
           ▼
   Status checks displayed on PR
   "All checks have passed"
           │
           ▼
   Merge button enabled ✅
   (nếu required checks đã pass)
```

### Commit Status API

```yaml
# Workflow tự động tạo status check khi chạy
# Tên check = tên job trong workflow

jobs:
  lint:                    # → Status check tên "lint"
    runs-on: ubuntu-latest
    steps: ...

  test:                    # → Status check tên "test"
    runs-on: ubuntu-latest
    steps: ...

  build:                   # → Status check tên "build"
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps: ...
```

### Set Status Thủ Công (Dùng GitHub Script)

```yaml
      - name: Set pending status
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.repos.createCommitStatus({
              owner: context.repo.owner,
              repo: context.repo.repo,
              sha: context.sha,
              state: 'pending',        # pending | success | failure | error
              target_url: `${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              description: 'Build in progress',
              context: 'custom/build-check'
            })
```

### Required Status Checks — Cấu Hình Trên GitHub UI

```
Repository → Settings → Branches → Add rule
  ├── Branch name pattern: main
  ├── ✅ Require status checks to pass before merging
  │     ├── ✅ Require branches to be up to date before merging
  │     └── Status checks that are required:
  │           ├── lint         ← Tên job trong workflow
  │           ├── test (18)    ← Job "test" với matrix node=18
  │           ├── test (20)    ← Job "test" với matrix node=20
  │           └── build
  └── ✅ Include administrators (áp dụng cả với admin)
```

---

## Branch Protection Rules

### Các Rule Quan Trọng

```
Repository Settings → Branches → Branch protection rules

┌─────────────────────────────────────────────────────────┐
│  Protect matching branches: main                        │
│                                                         │
│  ┌─ Pull Requests ─────────────────────────────────┐   │
│  │ ✅ Require a pull request before merging        │   │
│  │    ✅ Require approvals: 2                      │   │
│  │    ✅ Dismiss stale reviews when pushed         │   │
│  │    ✅ Require review from Code Owners           │   │
│  │    ✅ Require approval of most recent push      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─ Status Checks ─────────────────────────────────┐   │
│  │ ✅ Require status checks to pass                │   │
│  │    ✅ Require branches to be up to date         │   │
│  │    Status checks: lint, test, build             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─ Restrictions ──────────────────────────────────┐   │
│  │ ✅ Require conversation resolution              │   │
│  │ ✅ Require signed commits                       │   │
│  │ ✅ Require linear history                       │   │
│  │ ✅ Include administrators                       │   │
│  │ ✅ Restrict force pushes                        │   │
│  │ ✅ Restrict deletions                           │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Tự Động Hóa Branch Protection Qua API

```yaml
# Dùng trong workflow setup hoặc terraform
jobs:
  setup-protection:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.ADMIN_TOKEN }}
          script: |
            await github.rest.repos.updateBranchProtection({
              owner: context.repo.owner,
              repo: context.repo.repo,
              branch: 'main',
              required_status_checks: {
                strict: true,
                contexts: ['lint', 'test (18)', 'test (20)', 'build']
              },
              enforce_admins: true,
              required_pull_request_reviews: {
                required_approving_review_count: 2,
                dismiss_stale_reviews: true,
                require_code_owner_reviews: true
              },
              restrictions: null,     // null = không restrict ai
              allow_force_pushes: false,
              allow_deletions: false
            });
```

---

## Required Reviews

### Code Review Workflow

```yaml
# Quy trình PR với required reviews:
#
# 1. Developer mở PR
# 2. GitHub tự động request reviewers (từ CODEOWNERS)
# 3. CI chạy status checks
# 4. Reviewer để lại comments
# 5. Developer push fix → stale reviews dismissed
# 6. Reviewer approve lại
# 7. Tất cả required checks pass → Merge enabled
```

### Auto-request Reviewers Trong Workflow

```yaml
on:
  pull_request:
    types: [opened, ready_for_review]

jobs:
  request-review:
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            // Thêm reviewers dựa trên files thay đổi
            const { data: files } = await github.rest.pulls.listFiles({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });

            const reviewers = new Set();
            for (const file of files) {
              if (file.filename.startsWith('backend/')) {
                reviewers.add('backend-team-lead');
              }
              if (file.filename.startsWith('frontend/')) {
                reviewers.add('frontend-team-lead');
              }
            }

            if (reviewers.size > 0) {
              await github.rest.pulls.requestReviewers({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: context.issue.number,
                reviewers: [...reviewers]
              });
            }
```

### PR Size Check

```yaml
      - name: Check PR size
        uses: actions/github-script@v7
        with:
          script: |
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });

            const additions = pr.additions;
            const deletions = pr.deletions;
            const total = additions + deletions;

            if (total > 500) {
              core.warning(
                `PR has ${total} changed lines (${additions} additions, ${deletions} deletions). ` +
                `Consider breaking into smaller PRs for easier review.`
              );
            }
```

---

## Merge Strategies

GitHub hỗ trợ 3 merge strategies (chiến lược merge):

### 1. Merge Commit (Merge Thường)

```
main:    A─B─────────────M
                        /
feature:    C─D─E──────/

Tạo merge commit M với 2 parents.
Giữ nguyên lịch sử, thấy rõ nhánh.
✅ Phù hợp khi cần trace lịch sử đầy đủ.
```

### 2. Squash Merge (Gộp Thành Một Commit)

```
main:    A─B─────────────S
                         ↑
feature:    C─D─E    Squash → một commit S

Gộp tất cả commits của PR thành 1 commit trên main.
Lịch sử main sạch hơn.
✅ Phù hợp khi muốn "one PR = one commit".
```

### 3. Rebase Merge (Tái Nền)

```
main:    A─B──C'─D'─E'
              ↑
feature:    C─D─E   Rebase lên đầu main

Replay từng commit lên đầu main, không tạo merge commit.
Lịch sử tuyến tính.
✅ Phù hợp khi muốn linear history.
```

### Enforce Merge Strategy Qua CI

```yaml
# Kiểm tra squash merge đã được chọn
on:
  pull_request:
    types: [opened, edited]

jobs:
  check-merge-strategy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            // Kiểm tra PR title theo conventional commits (cần cho squash merge)
            const title = context.payload.pull_request.title;
            const conventionalPattern = /^(feat|fix|docs|style|refactor|test|chore|ci|perf|revert)(\(.+\))?: .+/;

            if (!conventionalPattern.test(title)) {
              core.setFailed(
                `PR title "${title}" does not follow Conventional Commits format.\n` +
                `Expected: feat|fix|docs|...: description\n` +
                `Example: feat(auth): add OAuth2 login`
              );
            }
```

---

## CODEOWNERS

CODEOWNERS — file định nghĩa ai là owner (chủ sở hữu) của từng phần code. GitHub tự động request review từ owners khi code trong vùng đó thay đổi.

### Vị Trí File

```
.github/CODEOWNERS        # Ưu tiên cao nhất
CODEOWNERS
docs/CODEOWNERS
```

### Cú Pháp

```
# CODEOWNERS syntax — tương tự .gitignore

# Mặc định owner cho mọi file
*                       @org/default-reviewers

# Thư mục cụ thể (trailing / không required)
/src/backend/           @org/backend-team

# File cụ thể
package.json            @org/frontend-lead @org/devops
*.tf                    @org/infrastructure-team

# Pattern với wildcard
src/**/*.test.ts        @org/qa-team
.github/workflows/      @org/devops @org/platform-team

# Nested pattern
docs/**/*.md            @org/tech-writers

# Theo extension toàn repo
**/*.go                 @golang-experts
**/*.java               @java-experts

# Loại trừ (không hỗ trợ negate pattern như .gitignore)
# → Dùng specific pattern thay thế
```

### Ví Dụ Thực Tế

```
# Người chịu trách nhiệm cuối cùng
*                           @cto

# Backend team review backend code
/src/api/                   @org/backend @backend-lead

# Frontend team
/src/web/                   @org/frontend @frontend-lead

# Infrastructure — ai thay đổi cũng cần DevOps review
*.tf                        @devops-team
*.yml                       @devops-team
Dockerfile                  @devops-team
/helm/                      @devops-team
.github/workflows/          @devops-team

# Security review cho auth code
/src/auth/                  @security-team @backend-lead
/src/api/middleware/        @security-team

# Data team cho database migrations
/migrations/                @data-team @backend-lead

# Docs team
/docs/                      @tech-writers
*.md                        @tech-writers
```

---

## Rulesets

Rulesets (Bộ Quy Tắc) — tính năng mới hơn Branch Protection Rules, hỗ trợ Organization-level và nhiều patterns.

### Tạo Ruleset Qua API

```yaml
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.ADMIN_TOKEN }}
          script: |
            await github.rest.repos.createRepoRuleset({
              owner: context.repo.owner,
              repo: context.repo.repo,
              name: 'protect-main',
              target: 'branch',
              enforcement: 'active',
              conditions: {
                ref_name: {
                  include: ['refs/heads/main', 'refs/heads/release/**'],
                  exclude: []
                }
              },
              rules: [
                { type: 'pull_request', parameters: {
                    required_approving_review_count: 2,
                    dismiss_stale_reviews_on_push: true,
                    require_code_owner_review: true,
                    require_last_push_approval: true
                }},
                { type: 'required_status_checks', parameters: {
                    required_status_checks: [
                      { context: 'lint' },
                      { context: 'test' },
                      { context: 'build' }
                    ],
                    strict_required_status_checks_policy: true
                }},
                { type: 'non_fast_forward' },      // Require linear history
                { type: 'deletion' },              // Ngăn xóa nhánh
                { type: 'required_signatures' }    // Require signed commits
              ]
            });
```

---

## Merge Queue

Merge Queue (Hàng Đợi Merge) — GitHub Enterprise / Teams feature — giải quyết "merge hell" khi nhiều PRs cùng pass CI nhưng conflict khi merge đồng thời.

### Vấn Đề Merge Queue Giải Quyết

```
Không có Merge Queue:
PR-A (pass CI on main@v1) ──→ merge → main@v2
PR-B (pass CI on main@v1) ──→ merge → main@v3 (có thể broken!)

Với Merge Queue:
PR-A vào queue → test PR-A on top of main@v1 → merge → main@v2
PR-B vào queue → test PR-B on top of main@v2 → merge → main@v3 (safe!)
```

### Workflow Kích Hoạt Bởi Merge Queue

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  merge_group:               # Kích hoạt khi PR vào merge queue
    types: [checks_requested]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - run: npm run build
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Tại sao cần "Require branches to be up to date"?

**Trả lời mẫu:**
Không có option này, PR-A có thể pass CI khi branch `main@v1`, nhưng khi merge sau khi PR-B đã merge vào main, tổng hợp code có thể broken. "Require up to date" bắt buộc developer rebase PR lên HEAD của main trước khi merge — đảm bảo CI test trên code tổng hợp thật sự.

**Trade-off:** Nhiều rebases hơn — nhưng đây là đúng behavior cho protected branches.

### Câu 2: Squash merge vs Rebase merge — chọn cái nào?

**Trả lời mẫu:**
- **Squash merge:** Tốt cho teams muốn lịch sử `main` cực kỳ sạch — "one feature = one commit". Trade-off: mất chi tiết commits của PR, khó bisect bug trong PR đó.
- **Rebase merge:** Lịch sử tuyến tính, giữ từng commit. Yêu cầu developer viết commit messages tốt cho từng commit trong PR.
- **Merge commit:** Thấy rõ context của PR, dễ revert cả PR bằng `git revert -m 1`. Trade-off: lịch sử "bận" với nhiều merge commits.

**Recommendation:** Squash for feature branches, merge commit for release branches (để revert dễ hơn).

### Câu 3: CODEOWNERS file đặt ở đâu và ảnh hưởng gì đến CI?

**Trả lời mẫu:**
CODEOWNERS ở `.github/CODEOWNERS`. Khi "Require review from Code Owners" được bật trong branch protection, GitHub tự động request review từ owners và chặn merge cho đến khi ít nhất một owner approve. CI workflow không tự đọc CODEOWNERS — đây là tính năng của GitHub platform. Kết hợp CODEOWNERS + required reviews tạo ra kiểm soát trách nhiệm rõ ràng: ai sở hữu code đó phải review thay đổi.

---

## 📂 Điều Hướng

- [← Build & Artifacts](4-build-artifacts.md)
- [↑ Quay lại README](README.md)
- [→ CD & Deployments](../03-cd-deployments/README.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
