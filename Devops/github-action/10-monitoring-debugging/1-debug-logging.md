# 1. Debug Logging — Gỡ Lỗi Qua Nhật Ký GitHub Actions

> Kỹ thuật debug workflow từ cơ bản đến nâng cao — sử dụng built-in logs, biến `ACTIONS_STEP_DEBUG`, và structured logging (ghi nhật ký có cấu trúc).

---

## 📋 Mục Lục

1. [Cấu Trúc Logs Trong GitHub Actions](#1-cấu-trúc-logs-trong-github-actions)
2. [ACTIONS_STEP_DEBUG](#2-actions_step_debug)
3. [ACTIONS_RUNNER_DEBUG](#3-actions_runner_debug)
4. [Workflow Commands — Lệnh Điều Khiển Output](#4-workflow-commands--lệnh-điều-khiển-output)
5. [Debug Techniques — Kỹ Thuật Gỡ Lỗi](#5-debug-techniques--kỹ-thuật-gỡ-lỗi)
6. [Structured Logging — Ghi Nhật Ký Có Cấu Trúc](#6-structured-logging--ghi-nhật-ký-có-cấu-trúc)
7. [Triage Logs Nhanh](#7-triage-logs-nhanh)
8. [Best Practices](#8-best-practices)

---

## 1. Cấu Trúc Logs Trong GitHub Actions

Mỗi workflow run (lần chạy workflow) có ba cấp logs:

```
Workflow Run
├── Job: build          ← mỗi job có log riêng
│   ├── Step: Checkout  ← mỗi step có log riêng
│   ├── Step: Test
│   └── Step: Build
└── Job: deploy
    ├── Step: Login
    └── Step: Push
```

### Truy Cập Logs

```
GitHub UI:
Actions tab → chọn workflow run → chọn job → xem từng step

Download logs:
Actions tab → workflow run → ⚙️ → "Download log archive"

GitHub CLI (gh — Giao Diện Dòng Lệnh GitHub):
gh run view <run-id> --log
gh run view <run-id> --log-failed   # chỉ logs của steps bị lỗi
```

### Retention Policy (Chính Sách Lưu Trữ Logs)

| Loại | Thời Gian Mặc Định | Tối Đa |
|---|---|---|
| Workflow logs | 90 ngày | 400 ngày |
| Artifacts | 90 ngày | 400 ngày |
| Cache | 7 ngày (không dùng) | — |

Thay đổi retention tại: **Settings → Actions → General → Artifact and log retention**

---

## 2. ACTIONS_STEP_DEBUG

`ACTIONS_STEP_DEBUG` — biến bật chế độ debug chi tiết cho từng step trong workflow.

### Kích Hoạt Qua UI

```
Actions tab
  → Chọn workflow run bị lỗi
  → "Re-run jobs"
  → Tích "Enable debug logging"
  → Re-run
```

### Kích Hoạt Qua Secret/Variable

```yaml
# Thêm secret hoặc variable vào repository:
# Tên: ACTIONS_STEP_DEBUG
# Giá trị: true

# GitHub tự động nhận biết và bật debug mode
```

### Kích Hoạt Trong Workflow (Tạm Thời)

```yaml
jobs:
  debug-job:
    runs-on: ubuntu-latest
    env:
      ACTIONS_STEP_DEBUG: true   # chỉ áp dụng cho job này
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
```

### Thông Tin Thêm Khi Bật Debug

Khi `ACTIONS_STEP_DEBUG=true`, mỗi step sẽ hiển thị thêm:

```
##[debug]Evaluating condition for step: 'Run tests'
##[debug]Evaluating: success()
##[debug]Evaluating success:
##[debug]=> true
##[debug]Result: true
##[debug]Starting: Run tests
##[debug]Working directory: /home/runner/work/myrepo/myrepo
##[debug]File: /home/runner/work/myrepo/myrepo
```

---

## 3. ACTIONS_RUNNER_DEBUG

`ACTIONS_RUNNER_DEBUG` — bật debug cho runner (máy chạy), bao gồm thông tin về:
- Kết nối với GitHub Actions service
- Khởi tạo môi trường
- Thực thi process
- Cấu hình runner

```yaml
# Kích hoạt tương tự ACTIONS_STEP_DEBUG
# Thêm secret: ACTIONS_RUNNER_DEBUG = true

# Hoặc trong workflow:
env:
  ACTIONS_RUNNER_DEBUG: true
```

### Khi Nào Dùng

| Biến | Dùng Khi |
|---|---|
| `ACTIONS_STEP_DEBUG` | Step bị lỗi, cần xem logic thực thi |
| `ACTIONS_RUNNER_DEBUG` | Runner không kết nối được, môi trường sai |
| Cả hai | Lỗi khó tái hiện, cần tối đa thông tin |

---

## 4. Workflow Commands — Lệnh Điều Khiển Output

GitHub Actions hỗ trợ các lệnh đặc biệt để format output trong logs.

### Nhóm Logs (Log Grouping)

```yaml
- name: Install dependencies
  run: |
    echo "::group::npm install"
    npm install
    echo "::endgroup::"

    echo "::group::npm audit"
    npm audit --audit-level=high
    echo "::endgroup::"
```

Kết quả trong UI: mỗi `::group::` tạo ra một phần có thể thu gọn (collapsible).

### Thông Báo Mức Độ (Annotations)

```yaml
- name: Check file
  run: |
    # Cảnh báo — xuất hiện ở Summary tab
    echo "::warning file=src/app.js,line=42::Deprecated API usage detected"

    # Lỗi — đánh dấu bước thất bại
    echo "::error file=src/app.js,line=10::Null pointer exception"

    # Notice — thông tin thuần
    echo "::notice::Deployment completed to staging"
```

### Ẩn Dữ Liệu Nhạy Cảm (Masking)

```yaml
- name: Fetch token
  run: |
    TOKEN=$(curl -s https://api.example.com/token | jq -r '.token')
    echo "::add-mask::$TOKEN"   # ẩn TOKEN khỏi mọi log sau đây
    echo "Token fetched: $TOKEN"  # hiển thị: "Token fetched: ***"
```

### Set Output và Environment Variables

```yaml
- name: Set version
  id: version
  run: |
    VERSION=$(cat package.json | jq -r '.version')
    echo "version=$VERSION" >> $GITHUB_OUTPUT       # set output
    echo "APP_VERSION=$VERSION" >> $GITHUB_ENV      # set env var
    echo "Build version: $VERSION" >> $GITHUB_STEP_SUMMARY  # thêm vào summary

- name: Use version
  run: echo "Deploying version ${{ steps.version.outputs.version }}"
```

---

## 5. Debug Techniques — Kỹ Thuật Gỡ Lỗi

### Kỹ Thuật 1 — In Context

```yaml
- name: Dump GitHub context
  run: echo '${{ toJSON(github) }}'

- name: Dump job context
  run: echo '${{ toJSON(job) }}'

- name: Dump steps context
  run: echo '${{ toJSON(steps) }}'

- name: Dump runner context
  run: echo '${{ toJSON(runner) }}'

- name: Dump all env vars
  run: env | sort
```

### Kỹ Thuật 2 — Kiểm Tra Môi Trường

```yaml
- name: Diagnostic check
  run: |
    echo "=== System Info ==="
    uname -a
    echo ""
    echo "=== User ==="
    whoami
    id
    echo ""
    echo "=== Working Directory ==="
    pwd
    ls -la
    echo ""
    echo "=== Tools Available ==="
    which node python3 docker kubectl helm 2>/dev/null || true
    echo ""
    echo "=== Network ==="
    curl -s --max-time 5 https://api.github.com/zen || echo "Network check failed"
```

### Kỹ Thuật 3 — Điều Kiện Debug

```yaml
- name: Debug on failure
  if: failure()   # chỉ chạy khi có step trước bị lỗi
  run: |
    echo "=== Debug info ==="
    cat /var/log/app.log 2>/dev/null || echo "No app log found"
    docker ps -a 2>/dev/null || true
    kubectl get events --sort-by='.lastTimestamp' 2>/dev/null || true
```

### Kỹ Thuật 4 — SSH Debug (Tmate)

```yaml
- name: Setup tmate session (chỉ dùng khi debug thủ công)
  if: ${{ failure() && github.event_name == 'workflow_dispatch' }}
  uses: mxschmitt/action-tmate@v3
  with:
    limit-access-to-actor: true   # chỉ tác giả trigger mới SSH được
  timeout-minutes: 15
```

> **Cảnh báo bảo mật:** Không bao giờ bật tmate trên nhánh `main` hoặc production workflows — có thể lộ secrets.

### Kỹ Thuật 5 — Local Testing với act

```bash
# act — Công Cụ Chạy GitHub Actions Cục Bộ
# Cài đặt: https://github.com/nektos/act

# Chạy workflow cụ thể
act push

# Chạy job cụ thể
act -j build

# Truyền secrets
act -s GITHUB_TOKEN=xxx push

# Chạy với verbose output
act -v push

# List available jobs
act --list
```

---

## 6. Structured Logging — Ghi Nhật Ký Có Cấu Trúc

Structured logging giúp logs dễ parse và tìm kiếm hơn.

### JSON Logging Trong Bash

```yaml
- name: Structured log example
  run: |
    log() {
      local level=$1
      local message=$2
      echo "{\"level\":\"$level\",\"message\":\"$message\",\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"
    }

    log "info" "Starting build"
    log "info" "Building version: $APP_VERSION"

    if npm run build; then
      log "info" "Build succeeded"
    else
      log "error" "Build failed"
      exit 1
    fi
```

### Node.js Action với Structured Logs

```javascript
const core = require('@actions/core');

// Ghi log theo cấp độ
core.info('Starting deployment');
core.warning('Deprecated config detected');
core.error('Deployment failed');

// Debug (chỉ hiện khi ACTIONS_STEP_DEBUG=true)
core.debug('Internal variable: ' + JSON.stringify(config));

// Tạo group
core.startGroup('Installing dependencies');
// ... installation code
core.endGroup();

// Set output
core.setOutput('deployment-url', 'https://staging.example.com');

// Fail action
core.setFailed('Deployment failed: timeout');
```

---

## 7. Triage Logs Nhanh

### Checklist Khi Workflow Bị Lỗi

```
1. Nhìn Summary tab            ── job nào fail? (màu đỏ)
2. Mở job fail                 ── step nào fail? (nhìn dấu ×)
3. Đọc cuối logs step fail     ── error message cụ thể là gì?
4. Kiểm tra điều kiện          ── if: condition có đúng không?
5. Xem previous step output    ── step trước có set output đúng không?
```

### Lỗi Thường Gặp và Cách Debug

| Lỗi | Nguyên Nhân Thường Gặp | Cách Debug |
|---|---|---|
| `Process completed with exit code 1` | Command thất bại | Đọc logs phía trên error |
| `Resource not accessible by integration` | Thiếu permission | Kiểm tra `permissions` block |
| `Error: Input required and not supplied` | Thiếu input cho action | Kiểm tra `with:` block |
| `ENOENT: no such file` | File không tồn tại | Thêm `ls -la` để kiểm tra |
| `Rate limit exceeded` | Quá nhiều API calls | Kiểm tra GitHub API rate limits |
| `Unable to locate executable` | Tool chưa được cài | Thêm setup step |

---

## 8. Best Practices

### Nên Làm

```yaml
# Thêm tên rõ ràng cho mỗi step
- name: Run unit tests and collect coverage
  run: npm test -- --coverage

# Nhóm output dài
- name: Build application
  run: |
    echo "::group::Dependencies"
    npm ci
    echo "::endgroup::"
    
    echo "::group::Build"
    npm run build
    echo "::endgroup::"

# Luôn có step cleanup khi debug
- name: Cleanup temp files
  if: always()   # chạy dù success hay fail
  run: rm -rf /tmp/debug-*
```

### Không Nên Làm

```yaml
# KHÔNG print secrets ra logs
- run: echo "Token is ${{ secrets.MY_TOKEN }}"  # ❌ có thể lộ

# KHÔNG để debug code trong production workflows
- run: env  # ❌ in toàn bộ env có thể lộ secrets

# KHÔNG dùng set -x mà không lọc
- run: |
    set -x  # ❌ in mọi lệnh kể cả lệnh có secrets
    curl -H "Authorization: Bearer $TOKEN" https://api.example.com
```

### Quy Tắc Vàng

1. **Luôn đặt tên step rõ ràng** — giúp tìm lỗi nhanh hơn trong logs
2. **Dùng `::group::` cho output dài** — logs dễ đọc hơn
3. **Dùng `::add-mask::` cho data nhạy cảm** — trước khi print
4. **Bật debug chỉ khi cần** — debug logs tốn storage và có thể lộ thông tin
5. **Luôn có `if: failure()` step** — tự động thu thập thông tin khi lỗi

---

## 🔗 Liên Kết Liên Quan

- [2-workflow-notifications.md](./2-workflow-notifications.md) — Nhận thông báo khi lỗi
- [3-metrics-observability.md](./3-metrics-observability.md) — Theo dõi xu hướng lỗi
- [4-audit-logs.md](./4-audit-logs.md) — Audit trail cho compliance

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
