# Reusable Workflows — Workflow Tái Sử Dụng

> Đóng gói toàn bộ CI/CD pipeline thành một workflow có thể gọi lại từ nhiều repo — xây dựng internal platform workflow chuẩn hóa cho toàn tổ chức.

## 📚 Mục Lục

1. [Khái Niệm](#khái-niệm)
2. [Cấu Trúc Callee Workflow](#cấu-trúc-callee-workflow)
3. [Inputs — Giá Trị Đầu Vào](#inputs)
4. [Outputs — Giá Trị Đầu Ra](#outputs)
5. [Secrets — Bí Mật](#secrets)
6. [Caller Workflow — Cách Gọi](#caller-workflow)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
8. [Giới Hạn & Lưu Ý](#giới-hạn--lưu-ý)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm

**Reusable Workflow** (Workflow Tái Sử Dụng) là một workflow GitHub Actions được thiết kế để gọi từ workflow khác thông qua event đặc biệt `workflow_call`.

```
Organization/
├── platform-workflows/              ← Repo chứa shared workflows
│   └── .github/workflows/
│       ├── deploy-to-aws.yml        ← Callee (được gọi)
│       └── run-tests.yml
│
├── service-a/
│   └── .github/workflows/
│       └── ci.yml                   ← Caller (người gọi)
│
└── service-b/
    └── .github/workflows/
        └── ci.yml                   ← Caller (người gọi)
```

**Lợi ích:**
- **DRY** (Don't Repeat Yourself — Không Lặp Lại Chính Mình): Sửa một chỗ, áp dụng toàn bộ
- **Chuẩn hóa** (Standardization): Đảm bảo tất cả services dùng cùng quy trình deploy
- **Bảo mật tập trung** (Centralized Security): Secrets và policies quản lý từ một nơi
- **Dễ audit** (Auditability): Theo dõi ai gọi workflow nào, khi nào

---

## Cấu Trúc Callee Workflow

File được gọi (callee) phải có `workflow_call` trong `on:` block:

```yaml
# .github/workflows/deploy.yml — CALLEE (workflow được gọi)
name: Deploy to Environment

on:
  workflow_call:              # Event cho phép workflow này được gọi từ ngoài
    inputs:
      environment:
        description: 'Môi trường triển khai (staging / production)'
        type: string
        required: true
      image_tag:
        description: 'Docker image tag cần deploy'
        type: string
        required: true
      dry_run:
        description: 'Chỉ mô phỏng, không thực sự deploy'
        type: boolean
        default: false
    secrets:
      aws_role_arn:
        description: 'ARN của IAM Role để assume'
        required: true
      kubeconfig:
        required: false
    outputs:
      deploy_url:
        description: 'URL của môi trường vừa deploy'
        value: ${{ jobs.deploy.outputs.app_url }}

# Permissions áp dụng cho workflow này
permissions:
  id-token: write      # Cần cho OIDC (OpenID Connect — Xác Thực Không Mật Khẩu)
  contents: read

jobs:
  deploy:
    name: Deploy — ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}   # GitHub Environment với protection rules
    outputs:
      app_url: ${{ steps.get-url.outputs.url }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.aws_role_arn }}
          aws-region: ap-southeast-1

      - name: Deploy (dry run check)
        if: ${{ !inputs.dry_run }}
        run: |
          echo "Deploying ${{ inputs.image_tag }} to ${{ inputs.environment }}..."
          # Lệnh deploy thực tế ở đây

      - name: Get deployment URL
        id: get-url
        run: echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

---

## Inputs

`inputs` cho phép caller truyền giá trị vào callee workflow. Hỗ trợ ba kiểu dữ liệu:

```yaml
on:
  workflow_call:
    inputs:
      # Kiểu string (chuỗi ký tự)
      app_name:
        description: 'Tên ứng dụng'
        type: string
        required: true

      # Kiểu string với giá trị mặc định
      region:
        description: 'AWS Region'
        type: string
        required: false
        default: 'ap-southeast-1'

      # Kiểu boolean (đúng/sai)
      run_tests:
        description: 'Có chạy test không?'
        type: boolean
        default: true

      # Kiểu number (số)
      timeout_minutes:
        description: 'Timeout tính bằng phút'
        type: number
        default: 30
```

**Truy cập trong jobs:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "App: ${{ inputs.app_name }}"
      - run: echo "Region: ${{ inputs.region }}"
      - if: ${{ inputs.run_tests }}
        run: npm test
      - timeout-minutes: ${{ inputs.timeout_minutes }}
        run: ./long-running-script.sh
```

---

## Outputs

`outputs` cho phép callee trả về giá trị cho caller:

```yaml
# CALLEE — định nghĩa outputs
on:
  workflow_call:
    outputs:
      version:
        description: 'Phiên bản được build'
        value: ${{ jobs.build.outputs.app_version }}
      artifact_url:
        description: 'URL tải artifact'
        value: ${{ jobs.build.outputs.download_url }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      app_version: ${{ steps.version.outputs.tag }}      # Job output lấy từ step output
      download_url: ${{ steps.upload.outputs.artifact-url }}

    steps:
      - name: Get version
        id: version
        run: echo "tag=$(git describe --tags --abbrev=0)" >> $GITHUB_OUTPUT

      - name: Upload artifact
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: app-build
          path: dist/
```

```yaml
# CALLER — sử dụng outputs
jobs:
  build:
    uses: org/platform/.github/workflows/build.yml@main

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Version: ${{ needs.build.outputs.version }}"
          echo "Artifact: ${{ needs.build.outputs.artifact_url }}"
```

---

## Secrets

Callee phải khai báo tường minh secrets nào nó cần:

```yaml
# CALLEE — khai báo secrets
on:
  workflow_call:
    secrets:
      npm_token:
        description: 'NPM Registry token'
        required: true
      slack_webhook:
        description: 'Slack Webhook URL cho thông báo'
        required: false   # Optional secret

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Publish to NPM
        env:
          NODE_AUTH_TOKEN: ${{ secrets.npm_token }}   # Dùng secret từ caller
        run: npm publish

      - name: Notify Slack
        if: ${{ secrets.slack_webhook != '' }}
        run: |
          curl -X POST "${{ secrets.slack_webhook }}" \
            -d '{"text": "Published successfully!"}'
```

### secrets: inherit — Kế Thừa Toàn Bộ Secrets

```yaml
# CALLER — truyền tất cả secrets
jobs:
  deploy:
    uses: org/platform/.github/workflows/deploy.yml@main
    secrets: inherit   # Truyền toàn bộ secrets từ caller vào callee
    with:
      environment: staging
```

> **Cảnh báo bảo mật:** `secrets: inherit` tiện nhưng kém an toàn — callee nhận được MỌI secret của caller, kể cả những secret không cần thiết. Ưu tiên khai báo tường minh theo nguyên tắc least-privilege (quyền tối thiểu).

---

## Caller Workflow

Cú pháp gọi reusable workflow:

```yaml
# CALLER — .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # Job gọi reusable workflow — CÙng repo
  test:
    uses: ./.github/workflows/run-tests.yml    # Path tương đối trong cùng repo
    with:
      node_version: '20'
    secrets:
      npm_token: ${{ secrets.NPM_TOKEN }}

  # Job gọi reusable workflow — Repo KHÁC
  build:
    uses: myorg/platform/.github/workflows/build.yml@v2.1.0   # Tag cụ thể
    with:
      app_name: 'my-service'
    secrets:
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}

  # Job gọi reusable workflow — Chờ job trước hoàn thành
  deploy-staging:
    needs: [test, build]
    uses: myorg/platform/.github/workflows/deploy.yml@main
    with:
      environment: staging
      image_tag: ${{ needs.build.outputs.version }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_STAGING_ROLE_ARN }}
```

### Tham chiếu đến Reusable Workflow

| Cú pháp | Ý nghĩa |
|---|---|
| `./.github/workflows/deploy.yml` | Cùng repo, nhánh hiện tại |
| `org/repo/.github/workflows/deploy.yml@main` | Repo khác, nhánh main |
| `org/repo/.github/workflows/deploy.yml@v2` | Repo khác, tag v2 |
| `org/repo/.github/workflows/deploy.yml@abc1234` | Repo khác, commit SHA cụ thể |

> **Best Practice:** Pin theo tag (`@v2`) hoặc SHA (`@abc1234`) thay vì `@main` để tránh breaking changes bất ngờ.

---

## Ví Dụ Thực Tế

### Pattern: Central Deploy Platform

```yaml
# org/platform/.github/workflows/deploy-k8s.yml
# Callee: Deploy lên Kubernetes với approval gate

name: Deploy to Kubernetes

on:
  workflow_call:
    inputs:
      service_name:
        type: string
        required: true
      image_tag:
        type: string
        required: true
      namespace:
        type: string
        default: 'default'
      replicas:
        type: number
        default: 2
      environment:
        type: string
        required: true
    secrets:
      kubeconfig_base64:
        required: true
    outputs:
      rollout_status:
        value: ${{ jobs.deploy.outputs.status }}

permissions:
  contents: read
  deployments: write

jobs:
  validate:
    name: Validate inputs
    runs-on: ubuntu-latest
    steps:
      - name: Check image exists
        run: |
          docker manifest inspect ghcr.io/${{ github.repository }}/${{ inputs.service_name }}:${{ inputs.image_tag }}

  deploy:
    name: Deploy ${{ inputs.service_name }}
    needs: validate
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}   # Kích hoạt approval gate nếu configured
    outputs:
      status: ${{ steps.rollout.outputs.status }}

    steps:
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.kubeconfig_base64 }}" | base64 -d > /tmp/kubeconfig
          echo "KUBECONFIG=/tmp/kubeconfig" >> $GITHUB_ENV

      - name: Deploy with kubectl
        run: |
          kubectl set image deployment/${{ inputs.service_name }} \
            app=ghcr.io/${{ github.repository }}/${{ inputs.service_name }}:${{ inputs.image_tag }} \
            -n ${{ inputs.namespace }}
          kubectl scale deployment/${{ inputs.service_name }} \
            --replicas=${{ inputs.replicas }} \
            -n ${{ inputs.namespace }}

      - name: Wait for rollout
        id: rollout
        run: |
          if kubectl rollout status deployment/${{ inputs.service_name }} \
               -n ${{ inputs.namespace }} --timeout=5m; then
            echo "status=success" >> $GITHUB_OUTPUT
          else
            echo "status=failed" >> $GITHUB_OUTPUT
            exit 1
          fi

      - name: Cleanup kubeconfig
        if: always()
        run: rm -f /tmp/kubeconfig
```

```yaml
# service-a/.github/workflows/release.yml
# Caller: Service A gọi platform workflow

name: Release Service A

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        id: meta
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          docker build -t ghcr.io/${{ github.repository }}/service-a:$TAG .
          docker push ghcr.io/${{ github.repository }}/service-a:$TAG
          echo "version=$TAG" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    uses: myorg/platform/.github/workflows/deploy-k8s.yml@v3    # Pin theo tag
    with:
      service_name: service-a
      image_tag: ${{ needs.build.outputs.image_tag }}
      environment: staging
      namespace: staging
    secrets:
      kubeconfig_base64: ${{ secrets.STAGING_KUBECONFIG_B64 }}

  deploy-production:
    needs: deploy-staging
    uses: myorg/platform/.github/workflows/deploy-k8s.yml@v3
    with:
      service_name: service-a
      image_tag: ${{ needs.build.outputs.image_tag }}
      environment: production
      namespace: production
      replicas: 5
    secrets:
      kubeconfig_base64: ${{ secrets.PROD_KUBECONFIG_B64 }}
```

---

## Giới Hạn & Lưu Ý

### Giới Hạn Kỹ Thuật

| Giới Hạn | Chi Tiết |
|---|---|
| **Nesting levels** (Cấp lồng nhau) | Tối đa 4 cấp (A gọi B gọi C gọi D — không thể sâu hơn) |
| **Calling limit** (Giới hạn gọi) | Tối đa 20 reusable workflows trong một workflow |
| **Env variables** | Caller không thể override `env:` của callee |
| **Matrix** | Không thể dùng matrix strategy để gọi reusable workflow |

### Visibility — Ai Có Thể Gọi Ai?

| Trường Hợp | Điều Kiện |
|---|---|
| Cùng repo | Luôn hoạt động |
| Khác repo, cùng org | Repo callee phải là public HOẶC phải enable "Allow reuse" trong settings |
| Khác org | Repo callee phải là public |

### Lưu Ý Bảo Mật

- Callee chạy với context của **caller** repo — không phải repo chứa callee
- `github.repository` trong callee trả về tên caller repo
- Secrets của callee repo **không** tự động có sẵn trong callee workflow
- Dùng `permissions` block trong callee để giới hạn quyền tối thiểu

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa reusable workflow và composite action?**

A: Reusable workflow có thể chứa nhiều jobs, hỗ trợ GitHub environments với approval gates, và có `secrets:` block riêng. Composite action chỉ nhóm các steps lại, chạy trong cùng job với caller, không hỗ trợ environments hay multiple jobs. Chọn reusable workflow khi cần chia sẻ toàn bộ pipeline logic; chọn composite action khi chỉ cần gói gọn một nhóm steps.

**Q: `secrets: inherit` vs khai báo tường minh — nên dùng cái nào?**

A: Khai báo tường minh vì nguyên tắc least-privilege — callee chỉ nhận secrets thực sự cần. `secrets: inherit` tiện nhưng kém an toàn trong môi trường enterprise vì callee có thể vô tình nhận được credentials nhạy cảm không liên quan.

**Q: Giới hạn nesting của reusable workflows là bao nhiêu? Tại sao?**

A: Tối đa 4 cấp lồng nhau. GitHub giới hạn để tránh circular dependencies (phụ thuộc vòng) và để kiểm soát độ phức tạp của pipeline, đảm bảo khả năng debug và audit được.

**Q: Làm thế nào để truyền matrix values vào reusable workflow?**

A: Không thể dùng matrix strategy trực tiếp để gọi reusable workflow. Workaround: Dùng JSON array input và xử lý trong callee, hoặc gọi callee nhiều lần với các inputs khác nhau trong từng job riêng biệt.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
