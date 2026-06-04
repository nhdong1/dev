# Composite Actions — Action Kết Hợp

> Đóng gói nhiều steps thành một action tái sử dụng — viết bằng YAML, không cần Node.js hay Docker, phù hợp cho setup tasks và nhóm commands lặp đi lặp lại.

## 📚 Mục Lục

1. [Khái Niệm](#khái-niệm)
2. [Cấu Trúc action.yml](#cấu-trúc-actionyml)
3. [Inputs & Outputs](#inputs--outputs)
4. [Sử Dụng Trong Workflow](#sử-dụng-trong-workflow)
5. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
6. [So Sánh Với Reusable Workflow](#so-sánh-với-reusable-workflow)
7. [Giới Hạn & Lưu Ý](#giới-hạn--lưu-ý)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm

**Composite Action** (Action Kết Hợp) cho phép gộp nhiều steps thành một action YAML duy nhất. Không cần viết code JavaScript hay Dockerfile — chỉ cần YAML và shell commands.

### Khi Nào Dùng Composite Action?

- Setup môi trường lặp lại (Node.js + cache + install dependencies)
- Nhóm các bước kiểm tra (lint + type-check + test)
- Sequence commands phức tạp nhưng không cần logic lập trình
- Muốn đóng gói để share trong organization mà không cần Node.js runtime

### Vị Trí Lưu action.yml

```
# Cách 1: Gốc repo (action được dùng trực tiếp bằng tên repo)
my-action/
└── action.yml           → uses: myorg/my-action@v1

# Cách 2: Trong thư mục con (share nhiều actions từ một repo)
platform-actions/
├── setup-node/
│   └── action.yml       → uses: myorg/platform-actions/setup-node@v1
├── deploy-k8s/
│   └── action.yml       → uses: myorg/platform-actions/deploy-k8s@v1
└── notify-slack/
    └── action.yml       → uses: myorg/platform-actions/notify-slack@v1

# Cách 3: Local trong cùng repo (không share ra ngoài)
.github/
└── actions/
    └── setup-env/
        └── action.yml   → uses: ./.github/actions/setup-env
```

---

## Cấu Trúc action.yml

```yaml
# action.yml — file metadata bắt buộc của mọi custom action
name: 'Setup Node.js with Cache'                    # Tên hiển thị trong workflow logs
description: 'Cài đặt Node.js, khôi phục npm cache, và install dependencies'
author: 'Platform Team'                             # Tùy chọn

# Inputs — giá trị caller truyền vào
inputs:
  node-version:
    description: 'Phiên bản Node.js cần cài'
    required: false
    default: '20'
  working-directory:
    description: 'Thư mục chứa package.json'
    required: false
    default: '.'
  install-command:
    description: 'Lệnh install dependencies'
    required: false
    default: 'npm ci'

# Outputs — giá trị trả về cho caller
outputs:
  cache-hit:
    description: 'Có khôi phục được cache không (true/false)'
    value: ${{ steps.cache.outputs.cache-hit }}
  node-version:
    description: 'Phiên bản Node.js đã cài thực tế'
    value: ${{ steps.setup.outputs.node-version }}

# runs — bắt buộc, khai báo loại action
runs:
  using: 'composite'        # Bắt buộc là 'composite' cho composite action

  steps:
    # Mỗi step PHẢI có shell: nếu dùng run:
    - name: Setup Node.js
      id: setup
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
        cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json

    - name: Record cache status
      id: cache
      shell: bash
      run: |
        # Kiểm tra cache hit từ setup-node
        if [ "${{ steps.setup.outputs.cache-hit }}" = "true" ]; then
          echo "cache-hit=true" >> $GITHUB_OUTPUT
        else
          echo "cache-hit=false" >> $GITHUB_OUTPUT
        fi

    - name: Install dependencies
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: ${{ inputs.install-command }}

# Branding — hiển thị đẹp trên Marketplace (tùy chọn)
branding:
  icon: 'package'            # Tên icon từ Feather Icons
  color: 'green'             # black, blue, gray, green, orange, purple, red, yellow, white
```

---

## Inputs & Outputs

### Khai Báo Inputs

```yaml
inputs:
  required-input:
    description: 'Input bắt buộc'
    required: true             # Lỗi nếu caller không truyền

  optional-with-default:
    description: 'Input tùy chọn có giá trị mặc định'
    required: false
    default: 'default-value'

  optional-no-default:
    description: 'Input hoàn toàn tùy chọn'
    required: false
    # Không có default → giá trị là empty string nếu không truyền
```

**Trong steps, truy cập input bằng:** `${{ inputs.input-name }}`

### Khai Báo Outputs

Outputs phải liên kết đến output của một step cụ thể:

```yaml
outputs:
  result:
    description: 'Kết quả xử lý'
    value: ${{ steps.process.outputs.result }}   # Phải tham chiếu đến step output
```

Step tạo output:

```yaml
steps:
  - name: Process data
    id: process                                  # id là bắt buộc để outputs tham chiếu
    shell: bash
    run: |
      RESULT=$(./compute.sh)
      echo "result=$RESULT" >> $GITHUB_OUTPUT    # Ghi vào GITHUB_OUTPUT
```

---

## Sử Dụng Trong Workflow

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Gọi composite action local (cùng repo)
      - name: Setup environment
        id: setup
        uses: ./.github/actions/setup-node     # Path tương đối từ gốc repo
        with:
          node-version: '20'
          working-directory: packages/api

      # Dùng output của composite action
      - name: Log cache status
        run: echo "Cache hit: ${{ steps.setup.outputs.cache-hit }}"

      # Gọi composite action từ repo khác
      - name: Notify on failure
        if: failure()
        uses: myorg/platform-actions/notify-slack@v2
        with:
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
          message: "CI failed on ${{ github.ref }}"
```

---

## Ví Dụ Thực Tế

### 1. Setup Monorepo

```yaml
# .github/actions/setup-monorepo/action.yml
name: 'Setup Monorepo'
description: 'Checkout, cài Node.js, restore Turborepo cache, install dependencies'

inputs:
  node-version:
    default: '20'
  fetch-depth:
    description: 'Git fetch depth — 0 để lấy full history cho Turborepo'
    default: '2'

outputs:
  turbo-cache-hit:
    value: ${{ steps.turbo-cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - name: Checkout with history
      uses: actions/checkout@v4
      with:
        fetch-depth: ${{ inputs.fetch-depth }}

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Restore Turborepo cache
      id: turbo-cache
      uses: actions/cache@v4
      with:
        path: .turbo
        key: ${{ runner.os }}-turbo-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-turbo-

    - name: Install dependencies
      shell: bash
      run: npm ci --prefer-offline
```

### 2. Docker Build & Push

```yaml
# .github/actions/docker-build-push/action.yml
name: 'Docker Build and Push'
description: 'Build Docker image và push lên registry với multi-platform support'

inputs:
  image-name:
    required: true
  image-tag:
    required: true
  registry:
    default: 'ghcr.io'
  dockerfile:
    default: 'Dockerfile'
  context:
    default: '.'
  platforms:
    description: 'Platforms để build (ví dụ: linux/amd64,linux/arm64)'
    default: 'linux/amd64'
  push:
    description: 'Push image sau khi build'
    default: 'true'

outputs:
  image-digest:
    description: 'SHA256 digest của image đã push'
    value: ${{ steps.build.outputs.digest }}
  image-url:
    value: ${{ steps.set-url.outputs.url }}

runs:
  using: composite
  steps:
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: ${{ inputs.context }}
        file: ${{ inputs.dockerfile }}
        platforms: ${{ inputs.platforms }}
        push: ${{ inputs.push }}
        tags: ${{ inputs.registry }}/${{ inputs.image-name }}:${{ inputs.image-tag }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Set image URL
      id: set-url
      shell: bash
      run: |
        echo "url=${{ inputs.registry }}/${{ inputs.image-name }}:${{ inputs.image-tag }}" >> $GITHUB_OUTPUT
```

### 3. Slack Notification (Tái Sử Dụng Logic Thông Báo)

```yaml
# .github/actions/notify-slack/action.yml
name: 'Notify Slack'
description: 'Gửi thông báo Slack với format chuẩn — success, failure, hoặc info'

inputs:
  webhook-url:
    description: 'Slack Incoming Webhook URL'
    required: true
  status:
    description: 'Trạng thái: success | failure | info'
    required: false
    default: 'info'
  title:
    required: true
  message:
    required: false
    default: ''
  run-url:
    required: false
    default: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

runs:
  using: composite
  steps:
    - name: Set color based on status
      id: color
      shell: bash
      run: |
        case "${{ inputs.status }}" in
          success) echo "color=#36a64f" >> $GITHUB_OUTPUT ;;
          failure) echo "color=#ff0000" >> $GITHUB_OUTPUT ;;
          *)       echo "color=#439FE0" >> $GITHUB_OUTPUT ;;
        esac

    - name: Send Slack notification
      shell: bash
      run: |
        curl -s -X POST "${{ inputs.webhook-url }}" \
          -H 'Content-type: application/json' \
          --data '{
            "attachments": [{
              "color": "${{ steps.color.outputs.color }}",
              "title": "${{ inputs.title }}",
              "title_link": "${{ inputs.run-url }}",
              "text": "${{ inputs.message }}",
              "footer": "GitHub Actions • ${{ github.repository }}",
              "ts": '$(date +%s)'
            }]
          }'
```

---

## So Sánh Với Reusable Workflow

| Tiêu Chí | Composite Action | Reusable Workflow |
|---|---|---|
| **Đơn vị** | Một bước (step) trong job | Toàn bộ job(s) |
| **Chạy trên** | Job hiện tại của caller | Job riêng (runner mới) |
| **Startup overhead** | Không (cùng runner) | Có (cần khởi động job mới) |
| **Multiple jobs** | ❌ | ✅ |
| **GitHub Environments** | ❌ | ✅ (với approval gates) |
| **Secrets block riêng** | ❌ (dùng input thay thế) | ✅ |
| **Services (Docker)** | ❌ | ✅ |
| **Matrix strategy** | ✅ (inherited từ caller) | Có giới hạn |
| **GITHUB_TOKEN scope** | Inherited từ caller | Có thể set riêng |
| **Phù hợp cho** | Setup, utility steps | Deploy pipeline, multi-env |

---

## Giới Hạn & Lưu Ý

### Các Giới Hạn Quan Trọng

1. **Phải khai báo `shell:`** cho mỗi `run` step — nếu thiếu sẽ bị lỗi:
   ```yaml
   # ❌ Sai — thiếu shell
   - run: echo "hello"

   # ✅ Đúng
   - run: echo "hello"
     shell: bash
   ```

2. **Không hỗ trợ `services:`** — không thể chạy sidecar containers (PostgreSQL, Redis...)

3. **Không hỗ trợ `container:`** — không thể chỉ định container image cho toàn bộ job

4. **Không có `timeout-minutes` riêng** — bị giới hạn bởi timeout của caller job

5. **`continue-on-error` ở step level** — hoạt động bình thường, nhưng action không thể tự set `continue-on-error` cho caller job

### Điểm Cần Chú Ý Khi Debug

```yaml
# Lỗi hay gặp: dùng env var thay vì input
# ❌ Sai — $MY_INPUT không hoạt động trong composite action
- run: echo "$MY_INPUT"
  shell: bash

# ✅ Đúng — dùng ${{ inputs.my-input }}
- run: echo "${{ inputs.my-input }}"
  shell: bash
```

### Truyền Secrets Vào Composite Action

Composite action không có `secrets:` block — truyền secrets qua `inputs`:

```yaml
# action.yml
inputs:
  token:
    description: 'GitHub token hoặc PAT'
    required: true

runs:
  using: composite
  steps:
    - uses: some-action@v1
      with:
        token: ${{ inputs.token }}   # Sử dụng token từ input
```

```yaml
# workflow.yml — caller phải truyền secret qua with:
- uses: ./.github/actions/my-action
  with:
    token: ${{ secrets.MY_TOKEN }}   # ✅ Truyền secret vào input
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao composite action phải khai báo `shell:` cho mỗi run step?**

A: Composite action chạy trong context của caller job nhưng là một action độc lập — GitHub Actions không thể assume shell mặc định vì composite action có thể được chạy trên nhiều loại runner khác nhau (Linux, Windows, macOS). Khai báo tường minh `shell: bash` hoặc `shell: pwsh` đảm bảo hành vi nhất quán.

**Q: Làm sao truyền secret vào composite action?**

A: Composite action không có `secrets:` block. Truyền secrets qua `inputs:` từ caller workflow. Về bảo mật, GitHub vẫn mask (ẩn) các giá trị secret trong logs nếu chúng từ `${{ secrets.* }}` context.

**Q: Composite action có thể gọi reusable workflow không?**

A: Không trực tiếp. Composite action có thể gọi `uses: someorg/some-action@v1` nhưng không thể gọi `uses: someorg/some-repo/.github/workflows/some.yml@main` vì cú pháp `workflow_call` chỉ dùng trong jobs, không phải steps.

**Q: Khi nào nên ưu tiên composite action thay vì viết nhiều steps trực tiếp trong workflow?**

A: Khi cùng một nhóm steps xuất hiện ở ≥2 workflows (hoặc ≥2 jobs trong cùng workflow). Không nên tạo composite action chỉ vì muốn "clean up" khi steps chỉ dùng một lần — DRY không nên ưu tiên hơn readability trong trường hợp đó.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
