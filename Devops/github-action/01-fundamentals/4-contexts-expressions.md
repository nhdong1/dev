# Contexts & Expressions — Ngữ Cảnh và Biểu Thức

> Contexts (Ngữ Cảnh) là các object chứa thông tin về workflow run, runner, jobs, và nhiều thứ khác. Expressions (Biểu Thức) là cú pháp để truy cập và tính toán giá trị từ contexts. Đây là cơ chế mạnh nhất để viết workflow động và linh hoạt.

## 📋 Mục Lục

1. [Cú Pháp Expression](#cú-pháp-expression)
2. [Context: github](#context-github)
3. [Context: env](#context-env)
4. [Context: vars](#context-vars)
5. [Context: secrets](#context-secrets)
6. [Context: job](#context-job)
7. [Context: steps](#context-steps)
8. [Context: runner](#context-runner)
9. [Context: needs](#context-needs)
10. [Context: inputs](#context-inputs)
11. [Context: matrix](#context-matrix)
12. [Functions Tích Hợp](#functions-tích-hợp)
13. [Status Check Functions](#status-check-functions)

---

## Cú Pháp Expression

```yaml
${{ <expression> }}

# Dùng được ở:
name: Deploy ${{ github.ref_name }}           # giá trị field YAML
run: echo "Branch is ${{ github.ref_name }}"  # trong run: command
if: github.ref == 'refs/heads/main'           # trong if: (không cần ${{ }})
env:
  MY_VAR: ${{ secrets.API_KEY }}              # trong env: value
with:
  token: ${{ secrets.GITHUB_TOKEN }}          # trong with: (action inputs)
```

### Truy Cập Property

```yaml
# Dot notation (ký hiệu chấm)
${{ github.actor }}
${{ github.event.pull_request.number }}

# Bracket notation (ký hiệu ngoặc) — dùng khi key có ký tự đặc biệt
${{ github.event['pull_request']['number'] }}

# Wildcard — lấy tất cả phần tử của mảng/object
${{ github.event.pull_request.labels.*.name }}
# → ["bug", "enhancement", "help wanted"]
```

---

## Context: `github`

Context `github` chứa thông tin về workflow run và event kích hoạt.

### Thuộc Tính Quan Trọng

```yaml
steps:
  - name: GitHub context info
    run: |
      # Thông tin Repository
      echo "Repo: ${{ github.repository }}"           # owner/repo-name
      echo "Owner: ${{ github.repository_owner }}"    # owner
      echo "Repo name: ${{ github.event.repository.name }}" # repo-name
      echo "Workspace: ${{ github.workspace }}"       # /home/runner/work/repo/repo
      echo "Server URL: ${{ github.server_url }}"     # https://github.com

      # Thông tin Commit/Ref
      echo "SHA: ${{ github.sha }}"                   # commit SHA đầy đủ (40 ký tự)
      echo "Ref: ${{ github.ref }}"                   # refs/heads/main | refs/tags/v1.0
      echo "Ref name: ${{ github.ref_name }}"         # main | v1.0
      echo "Ref type: ${{ github.ref_type }}"         # branch | tag
      echo "Head ref: ${{ github.head_ref }}"         # PR source branch
      echo "Base ref: ${{ github.base_ref }}"         # PR target branch

      # Thông tin Người Dùng và Event
      echo "Actor: ${{ github.actor }}"               # người trigger workflow
      echo "Triggering actor: ${{ github.triggering_actor }}" # người thực sự bấm nút
      echo "Event: ${{ github.event_name }}"          # push | pull_request | schedule...

      # Thông tin Workflow Run
      echo "Run ID: ${{ github.run_id }}"             # ID của lần chạy (unique)
      echo "Run number: ${{ github.run_number }}"     # số thứ tự (1, 2, 3...)
      echo "Run attempt: ${{ github.run_attempt }}"   # lần thử (1 nếu không retry)
      echo "Workflow: ${{ github.workflow }}"         # tên workflow
      echo "Workflow ref: ${{ github.workflow_ref }}" # file path và ref

      # Job info
      echo "Job: ${{ github.job }}"                   # job ID
      echo "Action: ${{ github.action }}"             # action đang chạy

      # API
      echo "API URL: ${{ github.api_url }}"           # https://api.github.com
      echo "GraphQL: ${{ github.graphql_url }}"       # GraphQL endpoint
```

### github.event — Toàn Bộ Payload

```yaml
- name: Full event payload
  run: echo '${{ toJSON(github.event) }}'
  # In ra toàn bộ webhook payload — rất hữu ích khi debug
```

### Ví Dụ Thực Tế

```yaml
steps:
  # Tạo URL đến PR
  - name: PR URL
    run: |
      PR_URL="${{ github.server_url }}/${{ github.repository }}/pull/${{ github.event.pull_request.number }}"
      echo "PR: $PR_URL"

  # Tạo URL đến commit
  - name: Commit URL
    run: |
      COMMIT_URL="${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}"
      echo "Commit: $COMMIT_URL"

  # Lấy 7 ký tự đầu của SHA
  - name: Short SHA
    id: sha
    run: echo "short=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
    # → ${{ steps.sha.outputs.short }} = "abc1234"

  # Tag Docker image bằng branch name
  - name: Docker tag
    run: |
      BRANCH="${{ github.ref_name }}"
      TAG="${BRANCH//\//-}"    # Thay / bằng - (main → main, feature/foo → feature-foo)
      docker build -t myapp:$TAG .
```

---

## Context: `env`

Truy cập biến môi trường trong expressions.

```yaml
env:
  APP_NAME: myapp
  VERSION: 1.2.3

jobs:
  build:
    env:
      BUILD_TYPE: release

    steps:
      - name: Use env vars
        run: |
          # Trong run: dùng $VAR_NAME (shell variable)
          echo "App: $APP_NAME"
          echo "Version: $VERSION"

          # Hoặc dùng expression (cẩn thận — không dùng secrets qua expressions)
          echo "App: ${{ env.APP_NAME }}"

      - name: Conditional on env var
        if: env.BUILD_TYPE == 'release'
        run: echo "This is a release build"
```

> **Lưu Ý:** Dùng `$VAR_NAME` trong shell thay vì `${{ env.VAR_NAME }}` trong `run:` để tránh log secrets vô tình.

---

## Context: `vars`

Variables (Biến) được cấu hình trong GitHub UI (không phải secrets — giá trị không được mã hóa).

```yaml
# Cài đặt: Settings → Secrets and variables → Actions → Variables

steps:
  - name: Use variables
    run: |
      echo "App URL: ${{ vars.APP_URL }}"
      echo "Region: ${{ vars.AWS_REGION }}"
      echo "Environment: ${{ vars.ENVIRONMENT }}"

  - name: Conditional deployment
    if: vars.ENABLE_DEPLOYMENT == 'true'
    run: ./deploy.sh
```

**Scope của vars:**
- **Repository variables:** Chỉ trong repo đó
- **Organization variables:** Tất cả repos trong org (nếu được phép)
- **Environment variables:** Chỉ trong environment cụ thể

---

## Context: `secrets`

Secrets (Bí Mật) — giá trị được mã hóa, không hiển thị trong logs.

```yaml
steps:
  - name: Use secrets
    run: |
      echo "This won't show in logs: ${{ secrets.MY_SECRET }}"
      # → In ra: "This won't show in logs: ***"

  - name: Deploy with credentials
    run: ./deploy.sh
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### secrets.GITHUB_TOKEN

Token đặc biệt, tự động tạo ra cho mỗi workflow run:

```yaml
steps:
  - name: Create PR comment
    uses: actions/github-script@v7
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      script: |
        github.rest.issues.createComment({
          issue_number: context.issue.number,
          owner: context.repo.owner,
          repo: context.repo.repo,
          body: 'Deployment successful! 🚀'
        })
```

**Quyền mặc định của GITHUB_TOKEN:**
- `contents: read` — đọc code
- `metadata: read` — đọc metadata

Nhiều actions cần khai báo thêm permissions (xem [1-workflow-syntax.md](1-workflow-syntax.md#khóa-permissions)).

---

## Context: `job`

Thông tin về job đang chạy.

```yaml
steps:
  - name: Job status
    if: always()
    run: |
      echo "Job status: ${{ job.status }}"
      # → success | failure | cancelled

  - name: Check container ID
    run: echo "Container ID: ${{ job.container.id }}"

  # services context
  - name: Service host
    run: |
      echo "DB host: ${{ job.services.postgres.id }}"
      echo "DB ports: ${{ toJSON(job.services.postgres.ports) }}"
```

---

## Context: `steps`

Truy cập outputs và kết quả của các steps đã chạy.

```yaml
steps:
  - name: Generate version
    id: version                            # ID để tham chiếu sau
    run: |
      VERSION="1.0.${{ github.run_number }}"
      echo "number=$VERSION" >> $GITHUB_OUTPUT
      echo "tag=v$VERSION" >> $GITHUB_OUTPUT

  - name: Build with version
    run: |
      echo "Building version ${{ steps.version.outputs.number }}"
      docker build -t myapp:${{ steps.version.outputs.tag }} .

  - name: Check previous step result
    if: steps.version.outcome == 'success'
    run: echo "Version step succeeded"

  - name: Handle previous step failure
    if: steps.version.outcome == 'failure'
    run: echo "Version step failed, using default"
```

### Step Outputs vs Conclusion vs Outcome

```yaml
steps:
  - id: my-step
    run: exit 1
    continue-on-error: true

  - run: |
      echo "Outcome: ${{ steps.my-step.outcome }}"
      # → failure (kết quả thực tế của step)

      echo "Conclusion: ${{ steps.my-step.conclusion }}"
      # → success (vì continue-on-error: true → step "thành công" theo nghĩa workflow)
```

| Thuộc Tính | Ý Nghĩa |
|---|---|
| `outcome` | Kết quả thực tế: `success`, `failure`, `cancelled`, `skipped` |
| `conclusion` | Kết quả sau `continue-on-error`: `success` nếu `continue-on-error: true` |
| `outputs.<name>` | Giá trị được set qua `$GITHUB_OUTPUT` |

---

## Context: `runner`

Thông tin về runner đang thực thi job.

```yaml
steps:
  - name: Runner info
    run: |
      echo "OS: ${{ runner.os }}"                # Linux | Windows | macOS
      echo "Arch: ${{ runner.arch }}"            # X64 | ARM | ARM64
      echo "Name: ${{ runner.name }}"            # tên runner
      echo "Temp: ${{ runner.temp }}"            # /tmp (thư mục tạm)
      echo "Tool cache: ${{ runner.tool_cache }}" # đường dẫn tool cache
      echo "Debug: ${{ runner.debug }}"           # 1 nếu debug mode bật
```

### Dùng runner.os Để Handle Đa Nền Tảng

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}

    steps:
      - name: Cross-platform command
        run: |
          if [ "${{ runner.os }}" = "Windows" ]; then
            echo "Windows path: $env:USERPROFILE"
          else
            echo "Unix path: $HOME"
          fi
        shell: bash    # dùng bash kể cả trên Windows (Git Bash)

      - name: Install tool (Linux/macOS)
        if: runner.os != 'Windows'
        run: brew install jq || apt-get install -y jq
```

---

## Context: `needs`

Truy cập outputs và kết quả của các jobs phụ thuộc.

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.set-version.outputs.value }}
      artifact-url: ${{ steps.upload.outputs.artifact-url }}

    steps:
      - id: set-version
        run: echo "value=1.2.3" >> $GITHUB_OUTPUT

      - id: upload
        uses: actions/upload-artifact@v4
        with:
          name: app
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version: ${{ needs.build.outputs.version }}"

  deploy:
    needs: [build, test]    # phụ thuộc nhiều jobs
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Build status: ${{ needs.build.result }}"
          # → success | failure | cancelled | skipped

          echo "Test status: ${{ needs.test.result }}"

          echo "Artifact: ${{ needs.build.outputs.artifact-url }}"

  notify:
    needs: [deploy]
    if: always()    # chạy dù deploy thành công hay thất bại
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [ "${{ needs.deploy.result }}" = "success" ]; then
            echo "Deploy thành công!"
          else
            echo "Deploy thất bại!"
          fi
```

---

## Context: `inputs`

Dùng trong `workflow_dispatch` và `workflow_call`.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [staging, production]
      debug:
        type: boolean
        default: false

  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Debug mode: ${{ inputs.debug }}"

        env:
          TARGET_ENV: ${{ inputs.environment }}
```

---

## Context: `matrix`

Truy cập giá trị trong matrix strategy.

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
        include:
          - os: ubuntu-latest
            node: 20
            experimental: true    # thêm key tùy chỉnh
        exclude:
          - os: windows-latest
            node: 18

    runs-on: ${{ matrix.os }}
    name: Test Node ${{ matrix.node }} on ${{ matrix.os }}

    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - name: Experimental feature
        if: matrix.experimental == true
        run: npm run test:experimental

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: results-${{ matrix.os }}-node${{ matrix.node }}
          path: test-results/
```

---

## Functions Tích Hợp

### contains()

```yaml
# Kiểm tra string có chứa substring không
${{ contains('hello world', 'world') }}     # → true

# Kiểm tra mảng có chứa value không
${{ contains(github.event.pull_request.labels.*.name, 'bug') }}

# Kiểm tra nhiều labels
${{ contains(github.event.pull_request.labels.*.name, 'deploy') &&
    contains(github.event.pull_request.labels.*.name, 'staging') }}
```

### startsWith() / endsWith()

```yaml
${{ startsWith(github.ref, 'refs/tags/v') }}      # tag release
${{ startsWith(github.ref, 'refs/heads/release/') }} # release branch
${{ endsWith(github.ref_name, '-rc') }}            # release candidate
${{ endsWith(github.event.head_commit.message, '[skip ci]') }}
```

### format()

```yaml
# format(string, ...args) — {0}, {1}, {2} là placeholders
${{ format('Hello {0}, welcome to {1}!', github.actor, github.repository) }}
# → "Hello octocat, welcome to owner/repo!"

${{ format('{0}/commit/{1}', github.server_url + '/' + github.repository, github.sha) }}
```

### join()

```yaml
# join(array, separator) — nối mảng thành string
${{ join(github.event.pull_request.labels.*.name, ', ') }}
# → "bug, enhancement, help wanted"

${{ join(matrix.versions, ' | ') }}
# → "18 | 20 | 22"
```

### toJSON() / fromJSON()

```yaml
# toJSON — chuyển object thành JSON string
- run: echo '${{ toJSON(github.event) }}'          # debug full event
- run: echo '${{ toJSON(job) }}'                   # debug job context

# fromJSON — parse JSON string thành object
- id: parse
  run: echo "data={\"key\":\"value\"}" >> $GITHUB_OUTPUT

- run: echo "${{ fromJSON(steps.parse.outputs.data).key }}"
  # → "value"

# Dùng fromJSON để tạo mảng động trong matrix
jobs:
  setup:
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          echo 'matrix={"node":["18","20","22"]}' >> $GITHUB_OUTPUT

  test:
    needs: setup
    strategy:
      matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

### hashFiles()

```yaml
# hashFiles(pattern) — hash nội dung file, dùng cho cache key
${{ hashFiles('**/package-lock.json') }}
${{ hashFiles('go.sum') }}
${{ hashFiles('requirements*.txt') }}
${{ hashFiles('Cargo.lock') }}

# Kết hợp nhiều files
${{ hashFiles('go.mod', 'go.sum') }}
```

---

## Status Check Functions

Dùng trong `if:` để kiểm tra trạng thái:

```yaml
steps:
  - name: Always runs
    if: always()
    run: echo "This always runs"

  - name: Only on success (default)
    if: success()
    run: echo "Previous steps succeeded"

  - name: Only on failure
    if: failure()
    run: |
      echo "Something failed!"
      # Send Slack alert, create GitHub issue...

  - name: Only when cancelled
    if: cancelled()
    run: echo "Workflow was cancelled"
```

### Kết Hợp Status Check Với Điều Kiện Khác

```yaml
steps:
  # Chạy khi fail, nhưng chỉ trên main branch
  - name: Alert on main failure
    if: failure() && github.ref == 'refs/heads/main'
    run: ./scripts/alert-team.sh

  # Chạy kể cả khi fail, nhưng chỉ khi step cụ thể thất bại
  - id: tests
    run: npm test
    continue-on-error: true

  - name: Handle test failure
    if: steps.tests.outcome == 'failure'
    run: echo "Tests failed, sending report..."
```

---

## Availability — Context Khả Dụng Ở Đâu

| Context | Workflow Keys | Job Conditions | Step Conditions |
|---|---|---|---|
| `github` | ✅ | ✅ | ✅ |
| `env` | ✅ | ✅ | ✅ |
| `vars` | ✅ | ✅ | ✅ |
| `secrets` | ❌ | ✅ | ✅ |
| `needs` | ❌ | ✅ | ✅ |
| `job` | ❌ | ❌ | ✅ |
| `steps` | ❌ | ❌ | ✅ |
| `runner` | ❌ | ❌ | ✅ |
| `inputs` | ✅ | ✅ | ✅ |
| `matrix` | ❌ | ✅ | ✅ |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: `github.actor` và `github.triggering_actor` khác nhau như thế nào?**

A:
- `github.actor` — người/bot đã commit hoặc trigger action ban đầu
- `github.triggering_actor` — người thực sự bấm "re-run" hoặc trigger workflow lần này

Ví dụ: Dependabot push commit (`actor = dependabot[bot]`), nhưng bạn bấm "Re-run jobs" (`triggering_actor = your-username`).

**Q: Làm thế nào để truyền data từ step này sang step khác?**

A: Dùng `$GITHUB_OUTPUT`:
```bash
echo "key=value" >> $GITHUB_OUTPUT
```
Sau đó tham chiếu: `${{ steps.<step-id>.outputs.key }}`

Không dùng environment variables cho cross-step data vì scope bị giới hạn.

**Q: `fromJSON()` dùng để làm gì trong matrix strategy?**

A: Tạo dynamic matrix — số lượng và giá trị matrix cells được tính toán trong runtime, không cố định trong file YAML. Ví dụ: một job đầu query database để lấy danh sách environments cần deploy, sau đó các jobs sau dùng danh sách đó làm matrix.

---

## 📂 Điều Hướng

- [← Runners](3-runners.md)
- [→ Environment Variables](5-environment-variables.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
