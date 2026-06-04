# Cú Pháp YAML Workflow — GitHub Actions

> YAML (YAML Ain't Markup Language — Ngôn Ngữ Đánh Dấu Cấu Hình) là định dạng file duy nhất để viết workflow GitHub Actions. Nắm vững cú pháp là bước đầu tiên bắt buộc.

## 📋 Mục Lục

1. [Cấu Trúc File Tổng Quan](#cấu-trúc-file-tổng-quan)
2. [Khóa name:](#khóa-name)
3. [Khóa on: — Events](#khóa-on--events)
4. [Khóa env: — Biến Toàn Cục](#khóa-env--biến-toàn-cục)
5. [Khóa defaults:](#khóa-defaults)
6. [Khóa concurrency:](#khóa-concurrency)
7. [Khóa jobs:](#khóa-jobs)
8. [Cấu Hình Job](#cấu-hình-job)
9. [Cấu Hình Steps](#cấu-hình-steps)
10. [Điều Kiện if:](#điều-kiện-if)
11. [Expressions — Biểu Thức](#expressions--biểu-thức)

---

## Cấu Trúc File Tổng Quan

```yaml
# .github/workflows/example.yml

name: <tên workflow>              # (tùy chọn) Tên hiển thị trên GitHub UI

run-name: <tên run>               # (tùy chọn) Tên của mỗi lần chạy cụ thể
                                  # Hỗ trợ expressions: ${{ github.actor }}

on: <events>                      # (bắt buộc) Sự kiện kích hoạt

env:                              # (tùy chọn) Biến môi trường toàn cục
  KEY: value

defaults:                         # (tùy chọn) Cấu hình mặc định cho run:
  run:
    shell: bash
    working-directory: ./src

concurrency:                      # (tùy chọn) Kiểm soát chạy đồng thời
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:                      # (tùy chọn) Quyền GITHUB_TOKEN
  contents: read
  pull-requests: write

jobs:                             # (bắt buộc) Định nghĩa các jobs
  <job-id>:
    ...
```

---

## Khóa `name:`

```yaml
name: CI — Build & Test           # Hiển thị ở tab Actions trên GitHub

run-name: Deploy ${{ github.ref_name }} by ${{ github.actor }}
# → "Deploy main by octocat" — hiển thị ở mỗi workflow run cụ thể
```

---

## Khóa `on:` — Events

### Dạng rút gọn (single event)

```yaml
on: push
```

### Dạng danh sách (multiple events)

```yaml
on: [push, pull_request]
```

### Dạng đầy đủ (full syntax với filters)

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'        # glob pattern
    branches-ignore:
      - 'docs/**'
    paths:
      - 'src/**'
      - '*.go'
    paths-ignore:
      - '**.md'
    tags:
      - 'v*'

  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches:
      - main

  schedule:
    - cron: '0 2 * * 1'    # Thứ Hai hàng tuần lúc 2:00 AM UTC

  workflow_dispatch:         # Kích hoạt thủ công qua UI hoặc API
    inputs:
      environment:
        description: 'Môi trường triển khai'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Bật debug mode?'
        required: false
        type: boolean
        default: false

  workflow_call:             # Được gọi từ workflow khác (reusable)
    inputs:
      version:
        required: true
        type: string
    secrets:
      DEPLOY_KEY:
        required: true
    outputs:
      artifact-url:
        description: 'URL của artifact đã build'
        value: ${{ jobs.build.outputs.url }}
```

---

## Khóa `env:` — Biến Toàn Cục

```yaml
env:
  NODE_ENV: production
  API_BASE_URL: https://api.example.com

jobs:
  build:
    env:                         # env cấp job — ghi đè env toàn cục
      NODE_ENV: development
    steps:
      - name: Install
        env:                     # env cấp step — ưu tiên cao nhất
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npm install
```

**Thứ tự ưu tiên (cao → thấp):** Step env → Job env → Workflow env → Runner default env

---

## Khóa `defaults:`

```yaml
defaults:
  run:
    shell: bash                  # shell mặc định cho tất cả run: steps
    working-directory: ./backend # thư mục làm việc mặc định

jobs:
  test:
    defaults:                    # defaults cấp job — ghi đè defaults toàn cục
      run:
        working-directory: ./frontend
    steps:
      - run: npm test            # chạy trong ./frontend
```

---

## Khóa `concurrency:`

Concurrency (Đồng Thời) — kiểm soát số workflow run cùng tồn tại trong một group.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # → group = "CI Pipeline-refs/heads/main"
  # Mọi run có cùng group sẽ không chạy đồng thời

  cancel-in-progress: true
  # → Hủy run cũ khi run mới bắt đầu cùng group
  # false → Run mới chờ run cũ xong (hàng đợi)
```

**Ứng dụng thực tế:**

```yaml
# Tránh deploy trùng nhau lên production
concurrency:
  group: deploy-production
  cancel-in-progress: false     # Chờ deploy trước hoàn thành

# Hủy CI cũ khi push code mới
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

---

## Khóa `permissions:`

Giới hạn quyền của GITHUB_TOKEN — token tự động được cấp cho mỗi workflow run.

```yaml
permissions:
  actions: read | write | none
  checks: read | write | none
  contents: read | write | none
  deployments: read | write | none
  id-token: write               # Cần thiết cho OIDC
  issues: read | write | none
  packages: read | write | none
  pull-requests: read | write | none
  repository-projects: read | write | none
  security-events: read | write | none
  statuses: read | write | none
```

**Best practice — least privilege (quyền tối thiểu):**

```yaml
permissions:
  contents: read    # mặc định chỉ đọc

jobs:
  deploy:
    permissions:
      contents: write          # chỉ job này mới cần write
      deployments: write
```

---

## Khóa `jobs:`

### Cấu Trúc Job Đầy Đủ

```yaml
jobs:
  build:                                    # job-id: chỉ chứa [a-z, A-Z, 0-9, -, _]
    name: Build Application                 # tên hiển thị (tùy chọn)
    runs-on: ubuntu-latest                  # runner
    needs: [setup, lint]                    # phụ thuộc job khác
    if: github.ref == 'refs/heads/main'     # điều kiện chạy job
    timeout-minutes: 30                     # timeout (mặc định 360 phút)
    continue-on-error: false                # dừng workflow nếu job fail

    permissions:
      contents: read

    env:
      BUILD_VERSION: ${{ github.sha }}

    outputs:                                # outputs để job khác sử dụng
      artifact-name: ${{ steps.build.outputs.name }}

    strategy:                               # matrix strategy
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false                      # tiếp tục matrix khi một cell fail
      max-parallel: 4

    container:                              # chạy job trong Docker container
      image: node:20-alpine
      credentials:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASSWORD }}
      env:
        NODE_ENV: test
      ports:
        - 3000
      volumes:
        - /tmp:/tmp
      options: --cpus 1

    services:                              # service containers (database, cache...)
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - ...
```

---

## Cấu Hình Steps

### Step dùng `uses:` (gọi action)

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4              # action từ Marketplace
    with:
      ref: ${{ github.sha }}
      fetch-depth: 0                       # lấy toàn bộ git history
      token: ${{ secrets.GITHUB_TOKEN }}

  - name: Setup Node.js
    uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'                         # tự động cache node_modules

  - name: Local action
    uses: ./.github/actions/my-action     # action trong cùng repo

  - name: Action từ repo khác
    uses: my-org/my-action@v1             # action từ repo khác
```

### Step dùng `run:` (chạy lệnh shell)

```yaml
steps:
  - name: Single line
    run: npm test

  - name: Multi-line
    run: |
      echo "Step 1"
      npm install
      npm test

  - name: Với shell cụ thể
    shell: python
    run: |
      import sys
      print(f"Python {sys.version}")

  - name: PowerShell
    shell: pwsh
    run: |
      $version = node --version
      Write-Host "Node: $version"

  - name: Với working-directory
    working-directory: ./packages/api
    run: go build ./...
```

### Thuộc Tính Step

```yaml
steps:
  - name: Tên hiển thị                    # (tùy chọn nhưng nên có)
    id: my-step                            # ID để tham chiếu outputs
    uses: actions/checkout@v4
    if: success()                          # điều kiện chạy step
    continue-on-error: true               # tiếp tục dù step fail
    timeout-minutes: 5                    # timeout cho step
    env:
      MY_VAR: value
    with:
      param1: value1
```

---

## Điều Kiện `if:`

### Status Check Functions

```yaml
if: success()          # step/job trước thành công (mặc định)
if: failure()          # có step/job nào đó thất bại
if: cancelled()        # workflow bị hủy
if: always()           # luôn chạy dù kết quả nào

# Kết hợp
if: failure() && github.ref == 'refs/heads/main'
```

### Điều Kiện Thông Dụng

```yaml
# Chỉ chạy trên nhánh main
if: github.ref == 'refs/heads/main'

# Không chạy với PRs từ fork
if: github.event.pull_request.head.repo.full_name == github.repository

# Chạy khi label 'deploy' được thêm vào PR
if: contains(github.event.pull_request.labels.*.name, 'deploy')

# Kiểm tra actor (người trigger)
if: github.actor != 'dependabot[bot]'

# Kiểm tra event type
if: github.event_name == 'push'

# Kết hợp điều kiện
if: |
  github.event_name == 'push' &&
  github.ref == 'refs/heads/main' &&
  !contains(github.event.head_commit.message, '[skip ci]')
```

---

## Expressions — Biểu Thức

### Cú Pháp Cơ Bản

```yaml
${{ <expression> }}      # dùng trong giá trị YAML
$[ <expression> ]        # không dùng (không phải GitHub syntax)
```

### Toán Tử

```yaml
# So sánh
${{ github.ref == 'refs/heads/main' }}
${{ github.run_number > 10 }}
${{ github.actor != 'bot' }}

# Logic
${{ true && false }}              # → false
${{ true || false }}              # → true
${{ !false }}                     # → true

# Truy cập object
${{ github.event.pull_request.number }}
${{ env.MY_VAR }}
${{ secrets.API_KEY }}
${{ vars.APP_URL }}

# Index mảng
${{ github.event.commits[0].message }}
```

### Functions Thông Dụng

```yaml
# contains(haystack, needle)
${{ contains(github.event.pull_request.labels.*.name, 'bug') }}
${{ contains('hello world', 'world') }}

# startsWith / endsWith
${{ startsWith(github.ref, 'refs/tags/v') }}
${{ endsWith(github.ref, '-rc') }}

# format
${{ format('Hello {0}, you are {1}!', 'World', 42) }}

# join
${{ join(matrix.os, ', ') }}

# toJSON / fromJSON
${{ toJSON(github.event) }}

# hashFiles — hash nội dung file (dùng cho cache key)
${{ hashFiles('**/package-lock.json') }}
${{ hashFiles('go.sum', 'go.mod') }}
```

---

## Workflow Output — Truyền Data Giữa Jobs

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.get-version.outputs.value }}
      artifact-url: ${{ steps.upload.outputs.artifact-url }}

    steps:
      - name: Get version
        id: get-version
        run: |
          VERSION=$(cat VERSION)
          echo "value=$VERSION" >> $GITHUB_OUTPUT
          # GITHUB_OUTPUT — file đặc biệt để set step outputs

  deploy:
    needs: build
    steps:
      - name: Use version from build job
        run: |
          echo "Deploying version: ${{ needs.build.outputs.version }}"
          echo "Artifact: ${{ needs.build.outputs.artifact-url }}"
```

---

## Ví Dụ Workflow Hoàn Chỉnh

```yaml
name: Full CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

defaults:
  run:
    shell: bash

permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  test:
    name: Test (${{ matrix.node }})
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20]
      fail-fast: false

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm

      - run: npm ci

      - name: Run tests
        id: run-tests
        run: npm test -- --reporter=json > test-results.json
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-node-${{ matrix.node }}
          path: test-results.json
          retention-days: 7

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    outputs:
      artifact-name: ${{ steps.set-output.outputs.name }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci && npm run build

      - id: set-output
        run: echo "name=app-${{ github.sha }}" >> $GITHUB_OUTPUT

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.set-output.outputs.name }}
          path: dist/
          retention-days: 30
```

---

## ⚠️ Lỗi Phổ Biến Trong YAML

### 1. Indent sai (lỗi YAML parse)

```yaml
# ❌ Sai — steps phải indent dưới job
jobs:
  build:
  runs-on: ubuntu-latest    # thiếu indent
  steps:
    - run: echo "hi"

# ✅ Đúng
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "hi"
```

### 2. Quên `|` cho multiline run

```yaml
# ❌ Sai
- run: echo "line1"
  echo "line2"

# ✅ Đúng
- run: |
    echo "line1"
    echo "line2"
```

### 3. Dùng tab thay space

YAML không chấp nhận tab characters. Luôn dùng spaces (2 hoặc 4).

### 4. Thiếu `--` trong expressions

```yaml
# ❌ Sai — không có ${{ }}
if: github.ref == 'refs/heads/main'

# ✅ Đúng
if: ${{ github.ref == 'refs/heads/main' }}

# Cũng đúng — GitHub tự thêm ${{ }} khi dùng trực tiếp trong if:
if: github.ref == 'refs/heads/main'
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa `env:` cấp workflow, job và step?**

A: Có 3 cấp độ env, ưu tiên cao hơn sẽ ghi đè thấp hơn:
- Workflow env: áp dụng cho mọi job và step
- Job env: chỉ áp dụng trong job đó
- Step env: chỉ áp dụng trong step đó — ưu tiên cao nhất

**Q: Làm thế nào để truyền data giữa các steps trong cùng job?**

A: Dùng `$GITHUB_OUTPUT` — ghi `key=value` vào file này, sau đó tham chiếu qua `${{ steps.<id>.outputs.<key> }}`.

**Q: `timeout-minutes` áp dụng cho job hay từng step?**

A: Có thể đặt cả hai. Job-level `timeout-minutes` giới hạn toàn bộ job. Step-level `timeout-minutes` giới hạn step đó. Nếu step vượt quá timeout, nó fail và job cũng fail (trừ khi `continue-on-error: true`).

---

## 📂 Điều Hướng

- [← README](README.md)
- [→ Events & Triggers](2-events-triggers.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
