# Docker Actions — Docker Container Actions

> Đóng gói action trong Docker container — chạy bất kỳ ngôn ngữ lập trình nào, kiểm soát hoàn toàn môi trường runtime, phù hợp cho tools đặc biệt và scripts phức tạp.

## 📚 Mục Lục

1. [Khái Niệm](#khái-niệm)
2. [Cấu Trúc Dự Án](#cấu-trúc-dự-án)
3. [action.yml cho Docker Action](#actionyml-cho-docker-action)
4. [Dockerfile](#dockerfile)
5. [Entrypoint Script](#entrypoint-script)
6. [Giao Tiếp Với GitHub Actions](#giao-tiếp-với-github-actions)
7. [Sử Dụng Pre-built Image](#sử-dụng-pre-built-image)
8. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
9. [So Sánh Với JavaScript Action](#so-sánh-với-javascript-action)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm

**Docker Action** chạy code bên trong một Docker container — có thể viết bằng Python, Go, Ruby, Shell, hoặc bất kỳ ngôn ngữ nào có Docker image.

### Luồng Hoạt Động

```
GitHub Actions Runner
│
├── Pull Docker image (hoặc build từ Dockerfile)
│
├── Start container với:
│   ├── GITHUB_* env variables (được inject tự động)
│   ├── /github/workspace mounted (checkout directory)
│   └── inputs được truyền qua env variables INPUT_*
│
└── Run entrypoint
    ├── Xử lý logic
    ├── Ghi outputs vào $GITHUB_OUTPUT
    └── Exit code → success (0) hoặc failure (non-zero)
```

### Khi Nào Dùng Docker Action?

- Cần ngôn ngữ khác (Python, Go, Ruby) — không muốn dùng JavaScript
- Tools đặc biệt cần cài đặt trong container (kubectl cụ thể version, custom binaries)
- Logic phức tạp nhưng đội không quen JavaScript
- Cần reproducible environment — Docker đảm bảo nhất quán
- Script hiện có bằng Python/Shell muốn đóng gói thành action

### Hạn Chế

- **Chỉ chạy trên Linux runners** — Docker không chạy trên Windows/macOS GitHub-hosted runners
- **Startup chậm hơn** — cần pull image (lần đầu) và khởi động container
- **Image size** ảnh hưởng đến tốc độ — ưu tiên dùng alpine/slim base image

---

## Cấu Trúc Dự Án

```
my-docker-action/
├── action.yml          # Metadata
├── Dockerfile          # Định nghĩa container image
├── entrypoint.sh       # Script chạy khi container start (hoặc main.py, main.go...)
└── src/
    └── main.py         # Code chính (ví dụ Python)
```

---

## action.yml cho Docker Action

```yaml
name: 'Database Migration Checker'
description: 'Kiểm tra migration files có conflict trước khi merge'
author: 'Platform Team'

inputs:
  migrations-dir:
    description: 'Thư mục chứa migration files'
    required: false
    default: 'migrations'
  base-branch:
    description: 'Branch cơ sở để so sánh'
    required: false
    default: 'main'
  fail-on-conflict:
    description: 'Fail workflow nếu phát hiện conflict'
    required: false
    default: 'true'

outputs:
  has-conflicts:
    description: 'Có conflict không (true/false)'
  conflicting-files:
    description: 'Danh sách files bị conflict, phân cách bởi dấu phẩy'

runs:
  using: 'docker'                    # Khai báo đây là Docker action
  image: 'Dockerfile'                # Build từ Dockerfile trong cùng thư mục
  # image: 'docker://python:3.12-alpine'  # Hoặc dùng pre-built public image
  # image: 'docker://ghcr.io/myorg/my-action:v1'  # Hoặc private image
  args:                              # Đây là ARGUMENTS truyền vào entrypoint (không phải env vars)
    - ${{ inputs.migrations-dir }}
    - ${{ inputs.base-branch }}
  env:                               # Env vars bổ sung trong container
    FAIL_ON_CONFLICT: ${{ inputs.fail-on-conflict }}

branding:
  icon: 'database'
  color: 'orange'
```

---

## Dockerfile

```dockerfile
# Dùng base image nhỏ — alpine tiết kiệm thời gian pull
FROM python:3.12-alpine

# Cài thêm git (cần để so sánh branches)
RUN apk add --no-cache git

# Set working directory
WORKDIR /action

# Copy requirements trước để tận dụng Docker layer cache
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY src/ ./src/
COPY entrypoint.sh .

# Đảm bảo script có quyền thực thi
RUN chmod +x entrypoint.sh

# Entrypoint — lệnh chạy khi container start
ENTRYPOINT ["/action/entrypoint.sh"]
```

### Best Practices cho Dockerfile

```dockerfile
# ✅ Dùng specific version — tránh 'latest' để đảm bảo reproducibility
FROM python:3.12.3-alpine3.19

# ✅ Dùng non-root user để tăng bảo mật
RUN addgroup -S action && adduser -S action -G action
USER action

# ✅ Multi-stage build — giảm kích thước final image
FROM python:3.12-alpine AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt --target=/deps

FROM python:3.12-alpine AS runtime
COPY --from=builder /deps /deps
ENV PYTHONPATH=/deps
COPY src/ /action/src/
COPY entrypoint.sh /action/
RUN chmod +x /action/entrypoint.sh
ENTRYPOINT ["/action/entrypoint.sh"]
```

---

## Entrypoint Script

```bash
#!/bin/bash
# entrypoint.sh

set -euo pipefail    # -e: exit on error, -u: error on undefined vars, -o pipefail

# === NHẬN INPUTS ===
# Cách 1: Qua arguments (khi dùng args: trong action.yml)
MIGRATIONS_DIR="${1:-migrations}"
BASE_BRANCH="${2:-main}"

# Cách 2: Qua environment variables INPUT_* (GitHub tự inject)
# INPUT_MIGRATIONS_DIR sẽ có value của inputs.migrations-dir
MIGRATIONS_DIR="${INPUT_MIGRATIONS_DIR:-migrations}"
FAIL_ON_CONFLICT="${FAIL_ON_CONFLICT:-true}"

echo "Kiểm tra migrations trong: $MIGRATIONS_DIR"
echo "So sánh với branch: $BASE_BRANCH"

# === LOGIC CHÍNH ===
# Lấy danh sách files changed
git fetch origin "$BASE_BRANCH" --depth=50
CHANGED_FILES=$(git diff --name-only "origin/$BASE_BRANCH" HEAD -- "$MIGRATIONS_DIR" 2>/dev/null || true)

if [ -z "$CHANGED_FILES" ]; then
  echo "Không có migration files nào thay đổi"
  {
    echo "has-conflicts=false"
    echo "conflicting-files="
  } >> "$GITHUB_OUTPUT"
  exit 0
fi

# Kiểm tra conflict (ví dụ: cùng timestamp)
CONFLICTS=""
declare -A timestamps

while IFS= read -r file; do
  # Lấy timestamp từ tên file (ví dụ: 20240101_create_users.sql)
  timestamp=$(basename "$file" | grep -oE '^[0-9]+' || true)
  if [ -n "$timestamp" ]; then
    if [ -n "${timestamps[$timestamp]+_}" ]; then
      CONFLICTS="$CONFLICTS,$file"
      echo "⚠️  Conflict: $file (trùng timestamp với ${timestamps[$timestamp]})"
    else
      timestamps[$timestamp]="$file"
    fi
  fi
done <<< "$CHANGED_FILES"

CONFLICTS="${CONFLICTS#,}"    # Xóa dấu phẩy đầu tiên

# === SET OUTPUTS ===
{
  if [ -n "$CONFLICTS" ]; then
    echo "has-conflicts=true"
    echo "conflicting-files=$CONFLICTS"
  else
    echo "has-conflicts=false"
    echo "conflicting-files="
  fi
} >> "$GITHUB_OUTPUT"

# === JOB SUMMARY ===
{
  echo "## Migration Check Results"
  if [ -n "$CONFLICTS" ]; then
    echo "### ❌ Conflicts Detected"
    echo "Files: $CONFLICTS"
  else
    echo "### ✅ No Conflicts Found"
    echo "Tất cả migration files đều unique"
  fi
} >> "$GITHUB_STEP_SUMMARY"

# === EXIT CODE ===
if [ -n "$CONFLICTS" ] && [ "$FAIL_ON_CONFLICT" = "true" ]; then
  echo "::error::Phát hiện migration conflict — xem chi tiết ở trên"
  exit 1
fi

exit 0
```

---

## Giao Tiếp Với GitHub Actions

Docker action giao tiếp với runner qua các file đặc biệt và workflow commands:

```bash
# === OUTPUTS ===
echo "key=value" >> "$GITHUB_OUTPUT"
echo "multiline-key<<EOF" >> "$GITHUB_OUTPUT"
echo "line 1" >> "$GITHUB_OUTPUT"
echo "line 2" >> "$GITHUB_OUTPUT"
echo "EOF" >> "$GITHUB_OUTPUT"

# === ENVIRONMENT VARIABLES cho steps tiếp theo ===
echo "MY_VAR=my_value" >> "$GITHUB_ENV"

# === PATH ===
echo "/path/to/tools" >> "$GITHUB_PATH"

# === JOB SUMMARY ===
echo "## My Summary" >> "$GITHUB_STEP_SUMMARY"
echo "| Key | Value |" >> "$GITHUB_STEP_SUMMARY"
echo "|---|---|" >> "$GITHUB_STEP_SUMMARY"
echo "| Status | ✅ |" >> "$GITHUB_STEP_SUMMARY"

# === WORKFLOW COMMANDS (legacy nhưng vẫn dùng) ===
echo "::debug::Debug message"
echo "::notice::Notice message"
echo "::warning::Warning message"
echo "::error::Error message"
echo "::error file=file.py,line=10,col=5::Error at specific location"

# Group logs
echo "::group::My group title"
echo "This is inside the group"
echo "::endgroup::"

# Mask value trong logs
echo "::add-mask::sensitive-value"
```

### Python Entrypoint

```python
#!/usr/bin/env python3
# main.py — Entrypoint Python thay thế cho shell script

import os
import sys
import subprocess

def set_output(key: str, value: str) -> None:
    """Ghi output về GitHub Actions runner."""
    github_output = os.environ.get('GITHUB_OUTPUT', '')
    if github_output:
        with open(github_output, 'a') as f:
            f.write(f"{key}={value}\n")
    else:
        # Fallback cho local testing
        print(f"::set-output name={key}::{value}")

def set_failed(message: str) -> None:
    print(f"::error::{message}", file=sys.stderr)
    sys.exit(1)

def main() -> None:
    # Lấy inputs — GitHub inject qua env vars INPUT_<NAME>
    token = os.environ.get('INPUT_GITHUB-TOKEN', '')
    environment = os.environ.get('INPUT_ENVIRONMENT', 'staging')

    if not token:
        set_failed("INPUT_GITHUB-TOKEN là bắt buộc")

    print(f"Processing deployment for: {environment}")

    # Logic chính
    result = process_deployment(environment)

    set_output('status', 'success' if result else 'failed')
    set_output('environment', environment)

    if not result:
        set_failed("Deployment failed")

def process_deployment(env: str) -> bool:
    # Logic thực tế ở đây
    print(f"Deploying to {env}...")
    return True

if __name__ == '__main__':
    main()
```

---

## Sử Dụng Pre-built Image

```yaml
# action.yml — dùng image có sẵn thay vì build
runs:
  using: 'docker'
  image: 'docker://hashicorp/terraform:1.7.0'   # Public Docker Hub image
  args:
    - version
```

```yaml
# Hoặc dùng image từ private registry
runs:
  using: 'docker'
  image: 'docker://ghcr.io/myorg/my-tool:v2.1.0'
  env:
    REGISTRY_TOKEN: ${{ inputs.registry-token }}
```

**Lưu ý về pre-built image:**
- Pin theo digest (SHA) thay vì tag để đảm bảo bất biến: `docker://python@sha256:abc123...`
- Tag có thể bị overwrite — digest thì không bao giờ thay đổi
- Public images từ Docker Hub có rate limit — dùng GitHub Container Registry (ghcr.io) cho production

---

## Ví Dụ Thực Tế

### Action Kiểm Tra Terraform Security

```dockerfile
# Dockerfile
FROM alpine:3.19

RUN apk add --no-cache \
    curl \
    bash \
    unzip

# Cài tfsec — Terraform static analysis tool
ARG TFSEC_VERSION=1.28.4
RUN curl -LO "https://github.com/aquasecurity/tfsec/releases/download/v${TFSEC_VERSION}/tfsec-linux-amd64" && \
    chmod +x tfsec-linux-amd64 && \
    mv tfsec-linux-amd64 /usr/local/bin/tfsec

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# entrypoint.sh
set -euo pipefail

TERRAFORM_DIR="${INPUT_TERRAFORM_DIR:-terraform}"
SEVERITY="${INPUT_MINIMUM_SEVERITY:-HIGH}"
FORMAT="${INPUT_OUTPUT_FORMAT:-sarif}"
OUTPUT_FILE="tfsec-results.${FORMAT}"

echo "Scanning: $TERRAFORM_DIR"
echo "Min severity: $SEVERITY"

# Chạy tfsec
tfsec "$TERRAFORM_DIR" \
  --minimum-severity "$SEVERITY" \
  --format "$FORMAT" \
  --out "$OUTPUT_FILE" \
  --no-color || SCAN_EXIT=$?

ISSUE_COUNT=$(tfsec "$TERRAFORM_DIR" --minimum-severity "$SEVERITY" --format json 2>/dev/null | \
              python3 -c "import sys,json; data=json.load(sys.stdin); print(len(data.get('results', [])))" 2>/dev/null || echo "0")

{
  echo "issue-count=$ISSUE_COUNT"
  echo "output-file=$OUTPUT_FILE"
  echo "scan-passed=$( [ "${SCAN_EXIT:-0}" -eq 0 ] && echo 'true' || echo 'false' )"
} >> "$GITHUB_OUTPUT"

{
  echo "## Terraform Security Scan"
  echo "- **Directory:** \`$TERRAFORM_DIR\`"
  echo "- **Issues found:** $ISSUE_COUNT"
  echo "- **Min severity:** $SEVERITY"
} >> "$GITHUB_STEP_SUMMARY"

exit "${SCAN_EXIT:-0}"
```

---

## So Sánh Với JavaScript Action

| Tiêu Chí | Docker Action | JavaScript Action |
|---|---|---|
| **Ngôn ngữ** | Bất kỳ | JavaScript / TypeScript |
| **Startup time** | Chậm hơn (pull + start container) | Nhanh (chạy trực tiếp trên runner) |
| **OS hỗ trợ** | Linux only | Linux, Windows, macOS |
| **Environment control** | Hoàn toàn (Docker) | Phụ thuộc runner |
| **Bundle phức tạp** | ❌ (không cần bundle) | ✅ (cần ncc build) |
| **GitHub API** | Cần tự cài SDK | `@actions/github` tích hợp sẵn |
| **Phù hợp cho** | Tools đặc biệt, Python/Go/Ruby | API calls, data processing nhanh |
| **Marketplace** | ✅ | ✅ |
| **Reproducibility** | Cao (Docker đảm bảo) | Phụ thuộc Node.js version của runner |

---

## Câu Hỏi Phỏng Vấn

**Q: Docker action có thể chạy trên Windows runner không?**

A: Không. Docker action chỉ hỗ trợ Linux runners. Nếu cần chạy trên Windows hoặc macOS, phải dùng JavaScript/composite action. Đây là giới hạn quan trọng cần nhớ khi thiết kế cross-platform CI/CD.

**Q: Tại sao nên pin Docker image theo digest thay vì tag?**

A: Tag (ví dụ `python:3.12`) là mutable — có thể bị override bất cứ lúc nào. Digest (SHA256) là immutable — đảm bảo action luôn chạy đúng image đã test. Đây là best practice cho supply chain security (bảo mật chuỗi cung ứng).

**Q: Làm thế nào Docker action nhận inputs từ workflow?**

A: GitHub Actions inject inputs thành environment variables với prefix `INPUT_` và uppercase tên input. Ví dụ: input `migrations-dir` → env var `INPUT_MIGRATIONS-DIR`. Ngoài ra có thể dùng `args:` trong action.yml để truyền qua arguments.

**Q: Khi nào nên build Dockerfile local vs dùng pre-built image?**

A: Dùng pre-built image (publish lên ghcr.io) khi: image build chậm, action được gọi nhiều lần, muốn control version image độc lập với action code. Build local khi: action mới phát triển, image chỉ dùng cho action này, hoặc cần ci/cd đơn giản.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
