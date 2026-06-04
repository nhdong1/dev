# Environment Variables — Biến Môi Trường

> Environment Variables (Biến Môi Trường) trong GitHub Actions có nhiều nguồn khác nhau: biến mặc định của GitHub, biến tự định nghĩa, và biến đặc biệt để giao tiếp với runner. Hiểu đúng từng loại giúp tránh lỗi khó debug.

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Default Environment Variables](#default-environment-variables)
3. [Biến Tự Định Nghĩa](#biến-tự-định-nghĩa)
4. [Environment Files — Giao Tiếp Với Runner](#environment-files--giao-tiếp-với-runner)
5. [Thứ Tự Ưu Tiên](#thứ-tự-ưu-tiên)
6. [Best Practices](#best-practices)

---

## Tổng Quan

```
Nguồn Biến Môi Trường trong GitHub Actions
│
├── Default Variables (Biến Mặc Định)
│   └── GITHUB_*, RUNNER_* — tự động inject bởi GitHub
│
├── Custom Variables (Biến Tự Định Nghĩa)
│   ├── workflow-level env:
│   ├── job-level env:
│   └── step-level env:
│
├── Environment Files (File Môi Trường Đặc Biệt)
│   ├── $GITHUB_ENV — thêm biến cho steps sau
│   ├── $GITHUB_OUTPUT — set step outputs
│   ├── $GITHUB_PATH — thêm vào PATH
│   └── $GITHUB_STEP_SUMMARY — ghi summary
│
└── Secrets & Variables (từ GitHub Settings)
    ├── ${{ secrets.MY_SECRET }}
    └── ${{ vars.MY_VAR }}
```

---

## Default Environment Variables

Biến mặc định do GitHub tự động inject vào mọi job. Đây là nguồn thông tin đáng tin cậy nhất.

### Nhóm GITHUB_*

| Biến | Giá Trị Ví Dụ | Mô Tả |
|---|---|---|
| `GITHUB_REPOSITORY` | `owner/my-repo` | Tên đầy đủ của repository |
| `GITHUB_REPOSITORY_OWNER` | `owner` | Chủ repo (user hoặc org) |
| `GITHUB_WORKFLOW` | `CI Pipeline` | Tên workflow |
| `GITHUB_WORKFLOW_REF` | `.github/workflows/ci.yml@refs/heads/main` | Đường dẫn + ref của workflow file |
| `GITHUB_RUN_ID` | `1658821493` | ID unique của workflow run |
| `GITHUB_RUN_NUMBER` | `42` | Số thứ tự run (tăng dần) |
| `GITHUB_RUN_ATTEMPT` | `1` | Số lần thử (1 = lần đầu) |
| `GITHUB_JOB` | `build` | Job ID |
| `GITHUB_ACTION` | `__run` | Tên action đang chạy |
| `GITHUB_ACTION_PATH` | `/home/runner/work/_actions/...` | Đường dẫn action |
| `GITHUB_ACTOR` | `octocat` | Người trigger workflow |
| `GITHUB_TRIGGERING_ACTOR` | `octocat` | Người trigger run này (kể cả re-run) |
| `GITHUB_SHA` | `ffac537e...` | Commit SHA đầy đủ (40 ký tự) |
| `GITHUB_REF` | `refs/heads/main` | Full ref |
| `GITHUB_REF_NAME` | `main` | Tên ref rút gọn |
| `GITHUB_REF_TYPE` | `branch` | Loại ref: `branch` hoặc `tag` |
| `GITHUB_HEAD_REF` | `feature/my-feature` | Source branch của PR |
| `GITHUB_BASE_REF` | `main` | Target branch của PR |
| `GITHUB_EVENT_NAME` | `push` | Tên event kích hoạt |
| `GITHUB_EVENT_PATH` | `/home/runner/work/_temp/...` | Đường dẫn đến event payload JSON |
| `GITHUB_WORKSPACE` | `/home/runner/work/repo/repo` | Thư mục làm việc |
| `GITHUB_SERVER_URL` | `https://github.com` | URL GitHub server |
| `GITHUB_API_URL` | `https://api.github.com` | URL GitHub API |
| `GITHUB_GRAPHQL_URL` | `https://api.github.com/graphql` | URL GraphQL |
| `GITHUB_TOKEN` | `ghs_...` | Token tự động (giống secrets.GITHUB_TOKEN) |
| `GITHUB_ENV` | `/home/runner/work/_temp/_runner_file_commands/...` | Đường dẫn đến GITHUB_ENV file |
| `GITHUB_OUTPUT` | `/home/runner/work/_temp/_runner_file_commands/...` | Đường dẫn đến GITHUB_OUTPUT file |
| `GITHUB_PATH` | `/home/runner/work/_temp/_runner_file_commands/...` | Đường dẫn đến GITHUB_PATH file |
| `GITHUB_STEP_SUMMARY` | `/home/runner/work/_temp/_runner_file_commands/...` | Đường dẫn đến summary file |

### Nhóm RUNNER_*

| Biến | Giá Trị Ví Dụ | Mô Tả |
|---|---|---|
| `RUNNER_OS` | `Linux` | OS của runner: `Linux`, `Windows`, `macOS` |
| `RUNNER_ARCH` | `X64` | Architecture: `X64`, `ARM`, `ARM64` |
| `RUNNER_NAME` | `GitHub Actions 2` | Tên runner |
| `RUNNER_TEMP` | `/home/runner/work/_temp` | Thư mục tạm (bị xóa sau job) |
| `RUNNER_TOOL_CACHE` | `/opt/hostedtoolcache` | Cache cho tools (setup-node...) |
| `RUNNER_DEBUG` | `1` | `1` nếu debug mode bật |

### Sử Dụng Trong Script

```yaml
steps:
  - name: Use default variables
    run: |
      # Trong shell script — dùng $VAR_NAME trực tiếp
      echo "Repository: $GITHUB_REPOSITORY"
      echo "Branch: $GITHUB_REF_NAME"
      echo "SHA short: ${GITHUB_SHA::7}"
      echo "Run #: $GITHUB_RUN_NUMBER"
      echo "Actor: $GITHUB_ACTOR"

      # Tạo Docker tag từ branch name và run number
      TAG="${GITHUB_REF_NAME//\//-}-$GITHUB_RUN_NUMBER"
      echo "Docker tag: $TAG"
```

---

## Biến Tự Định Nghĩa

### Cú Pháp env: — 3 Cấp Độ

```yaml
env:                              # Cấp Workflow — áp dụng cho mọi job và step
  NODE_ENV: production
  API_BASE: https://api.example.com
  RETRY_COUNT: "3"                # Giá trị luôn là string trong env vars

jobs:
  build:
    env:                          # Cấp Job — ghi đè workflow env
      NODE_ENV: development
      BUILD_DIR: ./dist

    steps:
      - name: Install
        env:                      # Cấp Step — ưu tiên cao nhất
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
          SKIP_OPTIONAL: "true"
        run: npm install

      - name: Build
        run: |
          echo "NODE_ENV: $NODE_ENV"     # development (job-level)
          echo "API_BASE: $API_BASE"     # https://api.example.com (workflow-level)
          echo "BUILD_DIR: $BUILD_DIR"   # ./dist (job-level)
```

### Dynamic Environment Variables

```yaml
steps:
  - name: Set dynamic vars
    run: |
      # Thêm biến phụ thuộc vào điều kiện
      if [ "${{ github.ref_name }}" = "main" ]; then
        echo "DEPLOY_ENV=production" >> $GITHUB_ENV
        echo "REPLICA_COUNT=3" >> $GITHUB_ENV
      else
        echo "DEPLOY_ENV=staging" >> $GITHUB_ENV
        echo "REPLICA_COUNT=1" >> $GITHUB_ENV
      fi

  - name: Deploy
    run: |
      echo "Deploying to: $DEPLOY_ENV"
      echo "Replicas: $REPLICA_COUNT"
      # DEPLOY_ENV và REPLICA_COUNT có sẵn từ step trước
```

---

## Environment Files — Giao Tiếp Với Runner

Đây là cơ chế quan trọng nhất để truyền data giữa steps và đặt cấu hình runtime.

### $GITHUB_ENV — Đặt Biến Cho Steps Sau

```yaml
steps:
  - name: Set environment variable
    run: |
      echo "MY_VAR=hello" >> $GITHUB_ENV
      echo "ANOTHER_VAR=world" >> $GITHUB_ENV

      # Multiline value
      echo "MULTILINE<<EOF" >> $GITHUB_ENV
      echo "line 1" >> $GITHUB_ENV
      echo "line 2" >> $GITHUB_ENV
      echo "EOF" >> $GITHUB_ENV

  - name: Use the variable
    run: |
      echo "MY_VAR: $MY_VAR"           # hello
      echo "ANOTHER_VAR: $ANOTHER_VAR" # world
      echo "MULTILINE: $MULTILINE"     # line 1\nline 2
```

> **Lưu Ý:** Biến set qua `$GITHUB_ENV` có hiệu lực từ step **tiếp theo**, không phải step hiện tại.

### $GITHUB_OUTPUT — Set Step Outputs

```yaml
steps:
  - name: Generate outputs
    id: gen
    run: |
      VERSION="1.2.3"
      TIMESTAMP=$(date +%s)
      ARTIFACT_NAME="app-$VERSION-$TIMESTAMP"

      echo "version=$VERSION" >> $GITHUB_OUTPUT
      echo "artifact-name=$ARTIFACT_NAME" >> $GITHUB_OUTPUT
      echo "timestamp=$TIMESTAMP" >> $GITHUB_OUTPUT

      # Multiline output
      echo "changelog<<EOF" >> $GITHUB_OUTPUT
      git log --oneline HEAD~5..HEAD >> $GITHUB_OUTPUT
      echo "EOF" >> $GITHUB_OUTPUT

  - name: Use outputs
    run: |
      echo "Version: ${{ steps.gen.outputs.version }}"
      echo "Artifact: ${{ steps.gen.outputs.artifact-name }}"
      echo "Changelog: ${{ steps.gen.outputs.changelog }}"
```

### $GITHUB_PATH — Thêm Vào PATH

```yaml
steps:
  - name: Install custom tool
    run: |
      # Download và extract tool
      wget https://example.com/my-tool.tar.gz
      tar xzf my-tool.tar.gz -C $HOME/.local/bin/

      # Thêm thư mục vào PATH cho steps sau
      echo "$HOME/.local/bin" >> $GITHUB_PATH

  - name: Use custom tool
    run: my-tool --version    # có thể chạy vì đã trong PATH
```

### $GITHUB_STEP_SUMMARY — Tạo Job Summary

Job Summary (Tóm Tắt Job) hiển thị trong GitHub UI tại tab Actions sau khi workflow chạy xong.

```yaml
steps:
  - name: Run tests
    id: tests
    run: npm test --reporter=json > test-results.json
    continue-on-error: true

  - name: Generate summary
    if: always()
    run: |
      # Ghi Markdown vào summary
      echo "## 🧪 Test Results" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "| Status | Count |" >> $GITHUB_STEP_SUMMARY
      echo "|--------|-------|" >> $GITHUB_STEP_SUMMARY

      PASSED=$(cat test-results.json | jq '.numPassedTests')
      FAILED=$(cat test-results.json | jq '.numFailedTests')

      echo "| ✅ Passed | $PASSED |" >> $GITHUB_STEP_SUMMARY
      echo "| ❌ Failed | $FAILED |" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY

      if [ "$FAILED" -gt "0" ]; then
        echo "### Failed Tests" >> $GITHUB_STEP_SUMMARY
        cat test-results.json | jq -r '.testResults[].testFilePath' >> $GITHUB_STEP_SUMMARY
      fi

      echo "" >> $GITHUB_STEP_SUMMARY
      echo "**Commit:** \`$GITHUB_SHA\`" >> $GITHUB_STEP_SUMMARY
      echo "**Run:** #$GITHUB_RUN_NUMBER" >> $GITHUB_STEP_SUMMARY
```

**Kết quả:** GitHub hiển thị bảng đẹp trong tab Actions → Job → Summary.

---

## Thứ Tự Ưu Tiên

Khi có biến cùng tên từ nhiều nguồn, thứ tự ưu tiên (cao → thấp):

```
1. Step-level env:          (cao nhất)
2. Job-level env:
3. Workflow-level env:
4. $GITHUB_ENV (set bởi step trước)
5. Default GITHUB_* / RUNNER_* variables   (thấp nhất)
```

```yaml
env:
  MY_VAR: workflow-level        # thấp nhất

jobs:
  build:
    env:
      MY_VAR: job-level         # ghi đè workflow-level

    steps:
      - name: Check var
        env:
          MY_VAR: step-level    # ghi đè job-level
        run: echo $MY_VAR       # → "step-level"
```

---

## Windows và Cross-Platform

```yaml
# Linux/macOS: dùng $VAR hoặc ${VAR}
- run: echo "Hello $MY_VAR"

# Windows PowerShell: dùng $env:VAR
- run: echo "Hello $env:MY_VAR"
  shell: pwsh

# Windows cmd: dùng %VAR%
- run: echo Hello %MY_VAR%
  shell: cmd

# Cross-platform — dùng bash trên mọi OS (qua Git Bash trên Windows)
- run: echo "Hello $MY_VAR"
  shell: bash
```

---

## Best Practices

### 1. Không Hardcode Giá Trị Nhạy Cảm

```yaml
# ❌ Sai — token lộ trong YAML file
- run: curl -H "Authorization: Bearer abc123" https://api.example.com

# ✅ Đúng — dùng secrets
- run: curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

### 2. Dùng Shell Variables Thay Vì Expressions Trong run:

```yaml
# ❌ Tránh — expression được expand trước khi shell chạy → có thể log secret
- run: echo "Token: ${{ secrets.MY_SECRET }}"

# ✅ Đúng — shell variable không bị log
- run: echo "Token: $MY_SECRET"
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
```

### 3. Dùng GITHUB_OUTPUT Thay Vì set-output (Deprecated)

```yaml
# ❌ Deprecated (không dùng nữa)
- run: echo "::set-output name=value::$RESULT"

# ✅ Đúng
- run: echo "value=$RESULT" >> $GITHUB_OUTPUT
```

### 4. Đặt Tên Biến Rõ Ràng

```yaml
# ❌ Tên không rõ ràng
env:
  URL: https://api.example.com
  KEY: mykey123

# ✅ Tên rõ ràng
env:
  API_BASE_URL: https://api.example.com
  DEPLOY_KEY_ID: mykey123
```

### 5. Kiểm Tra Biến Trước Khi Dùng

```yaml
steps:
  - name: Validate required vars
    run: |
      if [ -z "$DEPLOY_TOKEN" ]; then
        echo "::error::DEPLOY_TOKEN is not set"
        exit 1
      fi
      if [ -z "$TARGET_ENV" ]; then
        echo "::error::TARGET_ENV is not set"
        exit 1
      fi
    env:
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
      TARGET_ENV: ${{ vars.TARGET_ENV }}
```

---

## Workflow Commands — Lệnh Workflow

Ngoài environment files, có thể dùng workflow commands qua `echo "::command::"` syntax:

```yaml
steps:
  - name: Workflow commands
    run: |
      # Debug message (chỉ hiện khi ACTIONS_STEP_DEBUG=true)
      echo "::debug::This is debug info"

      # Notice — hiện thị ở đầu job summary
      echo "::notice file=app.js,line=1::License is required"

      # Warning — màu vàng trong logs
      echo "::warning file=config.yml,line=10::Config value deprecated"

      # Error — màu đỏ, đánh dấu job fail (nhưng không dừng step)
      echo "::error file=main.go,line=25::Missing null check"

      # Nhóm log lines (có thể collapse trong UI)
      echo "::group::Installation Logs"
      npm install
      echo "::endgroup::"

      # Ẩn giá trị trong log (mask secret)
      TEMP_TOKEN=$(get-temp-token)
      echo "::add-mask::$TEMP_TOKEN"
      echo "Token: $TEMP_TOKEN"    # → "Token: ***"
```

---

## Ví Dụ Tổng Hợp

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

env:
  APP_NAME: my-app
  REGISTRY: ghcr.io

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}
      version: ${{ steps.version.outputs.value }}

    steps:
      - uses: actions/checkout@v4

      - name: Compute version
        id: version
        run: |
          VERSION="${GITHUB_REF_NAME}-${GITHUB_SHA::7}"
          echo "value=$VERSION" >> $GITHUB_OUTPUT
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Docker meta
        id: meta
        run: |
          TAG="${{ env.REGISTRY }}/${{ github.repository_owner }}/${{ env.APP_NAME }}:$VERSION"
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Build image
        run: docker build -t "${{ steps.meta.outputs.tag }}" .

      - name: Push image
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ${{ env.REGISTRY }} -u ${{ github.actor }} --password-stdin
          docker push "${{ steps.meta.outputs.tag }}"

      - name: Summary
        run: |
          echo "## Build Complete 🚀" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Image:** \`${{ steps.meta.outputs.tag }}\`" >> $GITHUB_STEP_SUMMARY
          echo "**Version:** \`$VERSION\`" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** \`$GITHUB_SHA\`" >> $GITHUB_STEP_SUMMARY

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy
        run: |
          IMAGE="${{ needs.build.outputs.image-tag }}"
          echo "Deploying $IMAGE to production"
          kubectl set image deployment/$APP_NAME app=$IMAGE
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG }}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa `env:` và `$GITHUB_ENV`?**

A:
- `env:` trong YAML — khai báo tĩnh trong file, giá trị biết trước khi chạy
- `$GITHUB_ENV` — đặt biến **trong runtime**, có thể set giá trị động tính toán trong lúc chạy (ví dụ: output từ lệnh, giá trị từ API). Biến set trong bước N có hiệu lực từ bước N+1.

**Q: Tại sao không nên dùng `${{ secrets.MY_SECRET }}` trực tiếp trong `run:`?**

A: Expression `${{ }}` được expand **trước khi shell chạy** — giá trị sẽ xuất hiện dưới dạng plain text trong command line và có thể bị log trong debug mode. Thay vào đó, inject secret vào `env:` của step và dùng shell variable `$MY_SECRET` — GitHub tự mask giá trị secrets trong logs, nhưng đây là lớp bảo vệ bổ sung.

**Q: `$GITHUB_OUTPUT` vs `$GITHUB_ENV` — khi nào dùng cái nào?**

A:
- `$GITHUB_OUTPUT` — để truyền data ra **ngoài step** (sang step khác hoặc job khác qua `needs.*.outputs`)
- `$GITHUB_ENV` — để đặt biến môi trường cho **steps còn lại trong cùng job** (tiện hơn vì không cần khai báo `id:` để tham chiếu)

---

## 📂 Điều Hướng

- [← Contexts & Expressions](4-contexts-expressions.md)
- [↑ Quay lại INDEX](../INDEX.md)
- [→ 02-ci-pipeline/](../02-ci-pipeline/)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
