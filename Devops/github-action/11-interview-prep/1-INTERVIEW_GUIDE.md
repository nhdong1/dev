# Top 20 Câu Hỏi Phỏng Vấn GitHub Actions

> Bộ câu hỏi và đáp án chi tiết cho phỏng vấn GitHub Actions — từ cơ bản đến nâng cao, kèm tips trả lời thực chiến.

---

## 📋 Mục Lục

1. [Nhóm Nền Tảng (Câu 1–5)](#nhóm-nền-tảng)
2. [Nhóm CI Pipeline (Câu 6–9)](#nhóm-ci-pipeline)
3. [Nhóm CD & Deployments (Câu 10–12)](#nhóm-cd--deployments)
4. [Nhóm Security (Câu 13–16)](#nhóm-security)
5. [Nhóm Nâng Cao (Câu 17–20)](#nhóm-nâng-cao)
6. [Câu Hỏi Behavioral](#câu-hỏi-behavioral)
7. [Câu Hỏi Bẫy Phổ Biến](#câu-hỏi-bẫy-phổ-biến)

---

## Nhóm Nền Tảng

### Câu 1: GitHub Actions là gì? Giải thích kiến trúc cốt lõi.

**Cấp độ:** Junior | **Tần suất hỏi:** ⭐⭐⭐⭐⭐

**Đáp án mẫu:**

GitHub Actions là nền tảng CI/CD (Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Phân Phối Liên Tục) được tích hợp trực tiếp vào GitHub, cho phép tự động hóa toàn bộ vòng đời phần mềm từ build, test đến deploy.

**Kiến trúc gồm 5 thành phần chính:**

```
Events (Sự Kiện) → Workflow → Job → Step → Action
```

- **Event** — Sự kiện kích hoạt: push, pull_request, schedule, workflow_dispatch
- **Workflow** — File YAML trong `.github/workflows/`, định nghĩa toàn bộ quá trình tự động hóa
- **Job** — Nhóm các steps chạy trên cùng một runner; nhiều jobs có thể chạy song song
- **Step** — Một tác vụ đơn lẻ: chạy lệnh shell hoặc gọi một action
- **Action** — Đơn vị tái sử dụng nhỏ nhất; có thể từ Marketplace hoặc tự viết
- **Runner** — Máy chủ thực thi workflow: GitHub-hosted (Ubuntu, macOS, Windows) hoặc self-hosted

**Ví dụ thực tế:**

```yaml
name: CI Pipeline
on: push                          # Event (Sự Kiện)
jobs:
  test:                           # Job
    runs-on: ubuntu-latest        # Runner
    steps:
      - uses: actions/checkout@v4 # Action (từ Marketplace)
      - run: npm test             # Step (lệnh shell)
```

**Tip trả lời:** Vẽ sơ đồ tư duy khi giải thích. Kết thúc bằng ví dụ thực tế từ dự án của bạn.

---

### Câu 2: Sự khác biệt giữa `on: push` và `on: pull_request` là gì?

**Cấp độ:** Junior | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

| Tiêu Chí | `on: push` | `on: pull_request` |
|---|---|---|
| **Kích hoạt khi** | Commit được push lên nhánh | PR được tạo, cập nhật, hoặc sync |
| **Context (Ngữ Cảnh)** | `github.ref` = nhánh được push | `github.head_ref` = nhánh nguồn của PR |
| **Permissions (Quyền)** | Đầy đủ quyền của nhánh | Hạn chế hơn với fork PRs (secrets không accessible) |
| **Dùng để** | Deploy khi merge vào main | Kiểm tra code trước khi merge |

```yaml
on:
  push:
    branches: [main, develop]     # Chỉ trigger trên nhánh này
  pull_request:
    branches: [main]              # Chỉ trigger khi PR target vào main
    types: [opened, synchronize, reopened]
```

**Điểm quan trọng:** Workflow triggered bởi `pull_request` từ fork không có quyền truy cập secrets — đây là biện pháp bảo mật. Muốn chạy sau khi PR được review thì dùng `pull_request_target` (nhưng cần cẩn thận với security).

**Tip trả lời:** Đề cập đến security implications với fork PRs — đây là điểm phân biệt candidate hiểu sâu.

---

### Câu 3: Làm thế nào để chia sẻ dữ liệu giữa các Jobs trong một Workflow?

**Cấp độ:** Junior-Mid | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

Có 3 cách chính để chia sẻ dữ liệu giữa jobs:

#### Cách 1: Job Outputs (Đầu Ra Job) — cho dữ liệu nhỏ (chuỗi, số)

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

  deploy:
    needs: build                  # needs — phụ thuộc job trước
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

#### Cách 2: Artifacts (Tệp Đầu Ra) — cho file/thư mục lớn

```yaml
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
```

#### Cách 3: Cache — cho dependencies không thay đổi thường xuyên

```yaml
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

**Điểm quan trọng:** Job outputs có giới hạn kích thước (1MB). Artifacts tốt hơn cho binary, build output. Cache tốt hơn cho dependencies.

---

### Câu 4: `GITHUB_TOKEN` là gì? Nó khác gì Personal Access Token (PAT)?

**Cấp độ:** Junior-Mid | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

`GITHUB_TOKEN` là token tự động được GitHub tạo ra cho mỗi workflow run — không cần cấu hình thủ công.

| Tiêu Chí | GITHUB_TOKEN | Personal Access Token (PAT) |
|---|---|---|
| **Tạo ra bởi** | GitHub tự động | Người dùng tạo thủ công |
| **Thời hạn** | Hết hạn khi workflow run kết thúc | Theo cấu hình (7 ngày–1 năm) |
| **Scope (Phạm Vi)** | Giới hạn ở repository hiện tại | Có thể truy cập nhiều repos |
| **Bảo mật** | Tự động rotate, không lộ | Rủi ro nếu bị lộ |
| **Dùng khi** | Tác vụ trong cùng repo | Cần truy cập cross-repo |

```yaml
jobs:
  create-release:
    permissions:
      contents: write             # Cần khai báo rõ scope cần dùng
    steps:
      - uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Sử dụng token tự động
```

**Best practice:** Luôn dùng `GITHUB_TOKEN` thay PAT khi có thể. Chỉ dùng PAT khi cần cross-repository access.

---

### Câu 5: Giải thích `needs`, `if`, và `continue-on-error` trong GitHub Actions.

**Cấp độ:** Junior-Mid | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

#### `needs` — Dependency giữa Jobs

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    needs: test                   # Build chỉ chạy nếu test thành công
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  deploy:
    needs: [test, build]          # Cần cả hai jobs hoàn thành
    runs-on: ubuntu-latest
```

#### `if` — Điều kiện thực thi

```yaml
steps:
  - run: npm test

  - name: Deploy to Production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  - name: Notify on Failure
    if: failure()                 # Chỉ chạy nếu step trước thất bại
    run: ./notify.sh
```

**Các hàm điều kiện quan trọng:**
- `success()` — true nếu tất cả steps trước thành công
- `failure()` — true nếu có step thất bại
- `always()` — luôn chạy kể cả khi có lỗi
- `cancelled()` — true nếu workflow bị cancel

#### `continue-on-error` — Cho phép step thất bại mà không dừng job

```yaml
steps:
  - name: Run optional lint check
    continue-on-error: true       # Lỗi ở đây không dừng workflow
    run: npx eslint .
```

---

## Nhóm CI Pipeline

### Câu 6: Thiết kế một CI pipeline hoàn chỉnh cho Node.js application.

**Cấp độ:** Mid | **Tần suất hỏi:** ⭐⭐⭐⭐⭐

**Đáp án mẫu:**

```yaml
name: CI Pipeline — Node.js

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'            # Tự động cache node_modules

      - run: npm ci               # npm ci — install sạch từ lockfile
      - run: npm run lint
      - run: npm run format:check

  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest
    needs: lint                   # Test chỉ chạy sau khi lint pass
    strategy:
      matrix:
        node-version: [18, 20, 22]  # Test đa phiên bản Node
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm test -- --coverage

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-node-${{ matrix.node-version }}
          path: coverage/

  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7       # Giữ artifact 7 ngày
```

**Giải thích thiết kế:**
1. **Lint trước** — fail nhanh cho lỗi style, tiết kiệm thời gian
2. **Matrix testing** — đảm bảo tương thích đa phiên bản Node.js
3. **Build sau test** — không build code chưa pass test
4. **Cache npm** — giảm thời gian install từ ~2 phút xuống ~15 giây
5. **Upload artifacts** — giữ lại coverage report và build output

---

### Câu 7: Làm thế nào để cache dependencies hiệu quả?

**Cấp độ:** Mid | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

**Cache key strategy (Chiến Lược Khóa Cache)** là chìa khóa để cache hoạt động đúng:

```yaml
# npm
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Python pip
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-

# Maven (Java)
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
```

**Nguyên tắc cache key:**
- **Chính xác** (`hashFiles`) → cache hit khi lockfile không đổi
- **Fallback** (`restore-keys`) → dùng cache gần nhất nếu exact key miss
- **OS-specific** (`runner.os`) → tránh dùng cache của Windows trên Linux

**Tip quan trọng:** `setup-node`, `setup-python`, `setup-java` có tham số `cache:` tích hợp sẵn — nên dùng thay vì cấu hình thủ công `actions/cache`.

---

### Câu 8: Matrix Strategy là gì? Cho ví dụ thực tế.

**Cấp độ:** Mid | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

Matrix Strategy (Chiến Lược Ma Trận) cho phép chạy cùng một job với nhiều tổ hợp tham số khác nhau — song song.

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest    # Không test Windows + Node 18
            node: 18
        include:
          - os: ubuntu-latest     # Thêm tham số bổ sung
            node: 20
            experimental: true
      fail-fast: false            # Không dừng khi một combination thất bại
      max-parallel: 4             # Tối đa 4 jobs chạy cùng lúc

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

**Kết quả:** 3 OS × 3 Node versions = 9 jobs, trừ 1 excluded = **8 jobs chạy song song**.

**Ứng dụng thực tế:**
- Test thư viện open-source đa môi trường
- Verify backward compatibility
- Build cho multiple architectures (amd64, arm64)

---

### Câu 9: Reusable Workflows khác Composite Actions như thế nào?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

| Tiêu Chí | Reusable Workflow | Composite Action |
|---|---|---|
| **Đơn vị** | Toàn bộ workflow (nhiều jobs) | Nhiều steps trong một job |
| **Trigger** | `workflow_call` | `uses:` trong một step |
| **Có thể có jobs riêng** | ✅ Có | ❌ Không |
| **Chạy trên runner riêng** | ✅ Có thể | ❌ Kế thừa runner của caller |
| **Dùng khi** | Tái sử dụng toàn bộ pipeline | Đóng gói nhóm steps |

**Reusable Workflow — `.github/workflows/deploy.yml`:**

```yaml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy-key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.deploy-key }}
```

**Composite Action — `actions/setup-project/action.yml`:**

```yaml
name: Setup Project
runs:
  using: composite
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    - run: npm ci
      shell: bash
```

**Rule of thumb (Quy Tắc Ngón Tay Cái):**
- Cần chạy trên runner độc lập, có environments, approvals → **Reusable Workflow**
- Chỉ đóng gói steps hay dùng → **Composite Action**

---

## Nhóm CD & Deployments

### Câu 10: Thiết kế CD pipeline với Staging và Production environments.

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐⭐⭐

**Đáp án mẫu:**

```yaml
name: CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=commit-

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging               # Environment với protection rules
      url: https://staging.example.com
    steps:
      - run: kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}

  integration-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e
        env:
          BASE_URL: https://staging.example.com

  deploy-production:
    needs: integration-test
    runs-on: ubuntu-latest
    environment:
      name: production            # Yêu cầu manual approval (Phê Duyệt Thủ Công)
      url: https://example.com
    steps:
      - run: kubectl set image deployment/app app=${{ needs.build.outputs.image-tag }}
        env:
          KUBECONFIG: ${{ secrets.PRODUCTION_KUBECONFIG }}
```

**Key design decisions (Quyết Định Thiết Kế Quan Trọng):**
- **Environment protection rules** — Production yêu cầu reviewer approval
- **E2E tests (Kiểm Thử Đầu Cuối) trên staging** trước khi production
- **Image tag từ git SHA** — traceability (khả năng truy vết) hoàn chỉnh
- **Sequential jobs** — không deploy production nếu staging test fail

---

### Câu 11: Blue/Green Deployment là gì? Implement thế nào với GitHub Actions?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

Blue/Green Deployment là chiến lược duy trì hai môi trường production giống hệt nhau:
- **Blue** — phiên bản hiện tại đang chạy
- **Green** — phiên bản mới được deploy

Traffic switch (Chuyển Traffic) chỉ xảy ra sau khi Green đã healthy.

```yaml
jobs:
  deploy-green:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Green environment
        run: |
          # Deploy phiên bản mới lên Green
          kubectl apply -f k8s/deployment-green.yaml

      - name: Wait for Green to be healthy
        run: |
          kubectl rollout status deployment/app-green --timeout=5m

      - name: Run smoke tests on Green
        run: |
          curl -f https://green.internal.example.com/health

      - name: Switch traffic to Green
        run: |
          # Chuyển load balancer (Bộ Cân Bằng Tải) sang Green
          kubectl patch service app-service \
            -p '{"spec":{"selector":{"version":"green"}}}'

      - name: Keep Blue for rollback (Giữ lại Blue để rollback)
        run: echo "Blue environment kept for 24h rollback window"
```

**Ưu điểm:**
- Zero downtime deployment (Triển Khai Không Gián Đoạn)
- Rollback (Quay Lại Phiên Bản Cũ) tức thì — chỉ cần switch traffic ngược lại
- Test trên môi trường production-identical trước khi chuyển traffic

**Nhược điểm:**
- Tốn gấp đôi infrastructure trong quá trình deploy
- Database migration (Di Chuyển Cơ Sở Dữ Liệu) phức tạp hơn

---

### Câu 12: Làm thế nào để xử lý Database Migrations trong CI/CD?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

Database migration là một trong những phần phức tạp nhất của CD pipeline.

**Nguyên tắc chính:**

1. **Backward compatible migrations** — migration mới phải tương thích với code cũ
2. **Expand-Contract pattern** — thêm column mới (expand) → deploy → xóa column cũ (contract)
3. **Never delete data immediately** — đánh dấu deprecated trước

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Run database migrations
        run: |
          # Chạy migration trước khi deploy application
          npm run db:migrate
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}

      - name: Verify migration success
        run: npm run db:status

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy application
        run: kubectl rollout restart deployment/app
```

**Pattern an toàn hơn với rollback:**

```yaml
steps:
  - name: Create migration checkpoint
    run: npm run db:checkpoint --name "before-v2.0"

  - name: Run migration
    run: npm run db:migrate
    
  - name: Verify migration
    id: verify
    run: npm run db:verify
    continue-on-error: true

  - name: Rollback migration if failed
    if: steps.verify.outcome == 'failure'
    run: npm run db:rollback --to "before-v2.0"
```

---

## Nhóm Security

### Câu 13: OIDC là gì? Tại sao nó tốt hơn lưu cloud credentials trong secrets?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐⭐⭐

**Đáp án mẫu:**

OIDC — OpenID Connect — là giao thức xác thực cho phép GitHub Actions chứng minh danh tính với cloud providers mà **không cần lưu long-lived credentials** (Thông Tin Xác Thực Tồn Tại Lâu Dài).

**Cách hoạt động:**

```
GitHub Actions                    AWS / GCP / Azure
     │                                    │
     │  1. Yêu cầu OIDC token            │
     │──────────────────────────────────>│
     │  2. Token được ký bởi GitHub      │
     │                                    │
     │  3. Gửi token để exchange lấy     │
     │     temporary credentials          │
     │──────────────────────────────────>│
     │  4. Trả về short-lived token      │
     │<──────────────────────────────────│
     │                                    │
     │  5. Dùng token để gọi cloud API   │
```

**Ví dụ với AWS:**

```yaml
jobs:
  deploy:
    permissions:
      id-token: write             # Bắt buộc để request OIDC token
      contents: read

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: ap-southeast-1
          # Không cần AWS_ACCESS_KEY_ID hoặc AWS_SECRET_ACCESS_KEY!

      - run: aws s3 sync dist/ s3://my-bucket/
```

**So sánh với lưu credentials:**

| | Long-lived Credentials | OIDC |
|---|---|---|
| **Rủi ro lộ** | Cao — nếu secrets bị leak, attacker có access vĩnh viễn | Thấp — token chỉ sống vài phút |
| **Rotation** | Thủ công, hay quên | Tự động — mỗi run là token mới |
| **Audit trail** | Khó biết ai dùng khi nào | Đầy đủ — tied to workflow run |
| **Setup** | Đơn giản hơn | Phức tạp hơn (cần cấu hình IAM Role) |

**Tip trả lời:** Đây là hot topic — chuẩn bị giải thích cả cấu hình IAM Trust Policy phía AWS.

---

### Câu 14: Supply chain attack trong GitHub Actions là gì? Cách phòng ngừa?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

Supply chain attack (Tấn Công Chuỗi Cung Ứng) xảy ra khi attacker chiếm quyền kiểm soát một action từ Marketplace và inject (Chèn) malicious code (Mã Độc).

**Ví dụ nguy hiểm:**

```yaml
# NGUY HIỂM — có thể bị tấn công nếu tag bị overwrite
- uses: some-org/some-action@v1

# AN TOÀN — pin bởi commit SHA, không thể bị thay đổi
- uses: some-org/some-action@a8b7c6d5e4f3  # v1.2.3
```

**Biện pháp phòng ngừa:**

```yaml
steps:
  # 1. Pin tất cả actions bởi SHA đầy đủ (40 ký tự)
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
  - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0

  # 2. Tạo workflow để tự động cập nhật SHA bởi Dependabot
```

**Cấu hình Dependabot cho GitHub Actions — `.github/dependabot.yml`:**

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: "chore(deps)"
```

**Các biện pháp khác:**
- **Dependency review action** — scan PR cho known vulnerabilities
- **Minimal permissions** — `permissions: read-all` làm default
- **Fork critical actions** — kiểm soát hoàn toàn source code
- **Review action source** trước khi dùng

---

### Câu 15: Permissions block hoạt động như thế nào trong GitHub Actions?

**Cấp độ:** Mid | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

`permissions` block kiểm soát scope của `GITHUB_TOKEN` — nguyên tắc **least-privilege** (Quyền Tối Thiểu).

```yaml
# Ở cấp workflow — áp dụng cho tất cả jobs
permissions:
  contents: read          # Đọc code
  packages: write         # Push lên GHCR (GitHub Container Registry)
  pull-requests: write    # Comment trên PR
  issues: read

jobs:
  test:
    # Override cho job cụ thể
    permissions:
      contents: read
      checks: write       # Tạo check runs
    steps:
      - run: npm test
```

**Các scopes quan trọng:**

| Scope | Read | Write | Dùng khi |
|---|---|---|---|
| `contents` | Đọc code | Push, tạo release | Build, deploy |
| `packages` | Pull image | Push image | Docker builds |
| `pull-requests` | Đọc PR | Comment, label | Code review bots |
| `issues` | Đọc issues | Tạo/comment | Automation |
| `id-token` | — | Request OIDC token | Cloud auth |
| `deployments` | Đọc | Tạo deployment | CD pipelines |

**Best practice:** Set `permissions: read-all` ở cấp workflow, sau đó override cụ thể cho từng job.

---

### Câu 16: Làm thế nào để handle secrets khi cần debug workflow?

**Cấp độ:** Mid | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

Đây là vấn đề phổ biến — GitHub Actions tự động mask (Che) secrets trong logs, nhưng vẫn có rủi ro.

**Những điều KHÔNG nên làm:**

```yaml
# NGUY HIỂM — in secret ra logs
- run: echo "Debug: ${{ secrets.API_KEY }}"  # GitHub mask nhưng vẫn rủi ro
```

**Cách debug an toàn:**

```yaml
# 1. Bật debug logging — thêm secrets trong repository settings
# ACTIONS_STEP_DEBUG = true
# ACTIONS_RUNNER_DEBUG = true

# 2. Kiểm tra secret có tồn tại (không lộ giá trị)
- name: Verify secrets exist
  run: |
    if [ -z "${{ secrets.API_KEY }}" ]; then
      echo "ERROR: API_KEY secret is not set"
      exit 1
    fi
    echo "API_KEY is set (length: ${#API_KEY})"
  env:
    API_KEY: ${{ secrets.API_KEY }}

# 3. Dùng masked outputs
- name: Set masked value
  run: |
    VALUE="${{ secrets.SOME_VALUE }}"
    echo "::add-mask::$VALUE"  # Mask giá trị này trong tất cả logs sau
    echo "VALUE=$VALUE" >> $GITHUB_ENV
```

**Tip quan trọng:** Không bao giờ bật debug mode (`ACTIONS_STEP_DEBUG=true`) trên production workflows vì nó log ra runner diagnostic info có thể chứa sensitive data.

---

## Nhóm Nâng Cao

### Câu 17: Giải thích Concurrency Groups và khi nào dùng?

**Cấp độ:** Mid-Senior | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

Concurrency Groups (Nhóm Đồng Thời) ngăn nhiều workflow runs cùng thực hiện tác vụ có thể conflict (Xung Đột).

```yaml
# Chỉ cho phép 1 deployment tới production tại một thời điểm
concurrency:
  group: production-deploy
  cancel-in-progress: false       # Không cancel run đang chạy — chờ xong rồi chạy tiếp

# Với PR — cancel run cũ khi có commit mới
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true        # Hủy CI cũ khi push commit mới lên PR
```

**Khi nào dùng:**
- **Deploy workflows** — tránh race condition (Điều Kiện Chạy Đua) khi 2 PRs merge gần nhau
- **PR CI** — tiết kiệm minutes bằng cách cancel CI cũ
- **Resource-intensive jobs** — giới hạn số lượng jobs chạy song song

---

### Câu 18: Self-hosted runners phù hợp với trường hợp nào? Trade-offs là gì?

**Cấp độ:** Senior | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

**Dùng Self-hosted Runners khi:**

| Use Case | Lý Do |
|---|---|
| Cần GPU hoặc ARM hardware đặc biệt | GitHub-hosted không cung cấp |
| Truy cập internal services (VPN, private network) | GitHub-hosted không trong network |
| Build/test cần >6 giờ | GitHub-hosted limit 6h/job |
| Cần persistent caching lớn (>10GB) | GitHub-hosted cache giới hạn |
| Compliance yêu cầu code không ra ngoài | Data sovereignty |
| Tiết kiệm chi phí ở quy mô lớn | Sau ~3000 phút/tháng thì self-hosted rẻ hơn |

**Trade-offs:**

```
GitHub-hosted Runners:               Self-hosted Runners:
✅ Zero maintenance                   ❌ Phải tự maintain, patch
✅ Fresh environment mỗi run          ❌ Có thể có state sót lại
✅ Managed security                   ❌ Tự quản lý security
✅ Nhiều OS và versions               ❌ Giới hạn bởi những gì bạn setup
❌ Limited customization              ✅ Hoàn toàn tùy chỉnh
❌ Có thể tốn tiền ở quy mô lớn     ✅ Infrastructure cost tự kiểm soát
❌ Network egress ra internet         ✅ Private network access
```

**Ephemeral Runners (Runner Tạm Thời) với ARC:**

```yaml
# Actions Runner Controller (ARC — Bộ Điều Khiển Runner) trên Kubernetes
# Tự động scale lên/xuống theo demand
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: github-runners
spec:
  template:
    spec:
      repository: org/repo
      ephemeral: true             # Runner bị xóa sau mỗi job — bảo mật hơn
```

---

### Câu 19: Làm thế nào để tối ưu chi phí GitHub Actions ở quy mô lớn?

**Cấp độ:** Senior | **Tần suất hỏi:** ⭐⭐⭐

**Đáp án mẫu:**

**Các kỹ thuật tối ưu chi phí:**

#### 1. Path Filtering — Chỉ chạy CI khi relevant files thay đổi

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package*.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

#### 2. Concurrency + Cancel-in-progress — Hủy CI cũ

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

#### 3. Hiệu quả hóa cache

```yaml
# Cache hit rate cao → ít thời gian install dependencies
- uses: actions/setup-node@v4
  with:
    cache: 'npm'
    # setup-node tự handle cache key tối ưu
```

#### 4. Sử dụng đúng runner size

```yaml
# Dùng ubuntu (2-core) cho simple tasks thay vì ubuntu-latest-8-core
runs-on: ubuntu-latest          # 2 cores, $0.008/min
# vs
runs-on: ubuntu-latest-16-core  # 16 cores, $0.064/min (8x đắt hơn)
```

#### 5. Chạy nặng trên self-hosted runners

```yaml
jobs:
  heavy-build:
    runs-on: self-hosted          # Không tốn GitHub Actions minutes
```

**Tip tính toán:** Track workflow duration trends bằng GitHub REST API `/repos/{owner}/{repo}/actions/runs` để phát hiện jobs nào tốn nhiều nhất.

---

### Câu 20: Thiết kế GitHub Actions cho một công ty có 50+ repositories.

**Cấp độ:** Senior | **Tần suất hỏi:** ⭐⭐⭐⭐

**Đáp án mẫu:**

Đây là system design question — trả lời theo 4 chiều: standardization, reusability, security, observability.

**Kiến trúc đề xuất:**

```
central-workflows repository
├── .github/workflows/
│   ├── ci-node.yml          # Reusable CI cho Node.js projects
│   ├── ci-python.yml        # Reusable CI cho Python projects
│   ├── deploy-k8s.yml       # Reusable deploy lên Kubernetes
│   ├── security-scan.yml    # Reusable security scanning
│   └── notify.yml           # Reusable notification
└── actions/
    ├── setup-standard/      # Composite action cho standard setup
    └── deploy-helper/       # Composite action cho deploy helpers
```

**Từng repo sử dụng reusable workflows:**

```yaml
# repo-a/.github/workflows/ci.yml
jobs:
  ci:
    uses: org/central-workflows/.github/workflows/ci-node.yml@main
    with:
      node-version: '20'
    secrets: inherit            # Kế thừa secrets từ calling repo
```

**Quản lý Security ở cấp Organization:**

```yaml
# Organization-level: Required workflows (Workflows Bắt Buộc)
# Settings → Actions → Required workflows
# → security-scan.yml phải pass trên tất cả repos
```

**Governance (Quản Trị):**
- **Required workflows** — bắt buộc security scan, license check
- **Organization secrets** — shared credentials (deploy keys, notification tokens)
- **Rulesets** — enforce nhánh protection policies
- **Audit logs** — track ai chạy gì, khi nào

**Tip trả lời:** Hỏi về scale (bao nhiêu developers?), tech stack mix, cloud provider, compliance requirements → thể hiện thinking process.

---

## Câu Hỏi Behavioral

### B1: Kể về lần bạn cải thiện một CI/CD pipeline. Bạn đạt được gì?

**Framework STAR:**

```
Situation: Pipeline cũ mất 45 phút/run, team phàn nàn chờ quá lâu
Task:      Giảm xuống dưới 15 phút mà không giảm chất lượng test
Action:    
  1. Phân tích bottleneck (điểm nghẽn) bằng workflow timing logs
  2. Thêm cache cho npm dependencies → giảm 8 phút
  3. Chạy unit tests song song với matrix strategy → giảm 12 phút
  4. Move integration tests sang scheduled workflow (chỉ chạy 3x/ngày)
Result:    Pipeline từ 45 phút → 18 phút, team satisfaction tăng,
           GitHub Actions cost giảm 40%
```

---

### B2: Bạn đã xử lý deployment failure (Lỗi Triển Khai) như thế nào?

```
Situation: Deploy production lúc 9PM, sau 5 phút nhận alert error rate spike
Task:      Rollback nhanh nhất có thể, tìm nguyên nhân
Action:    
  1. Trigger rollback workflow ngay lập tức (manual dispatch)
  2. Thông báo team qua Slack alert tích hợp trong workflow
  3. Kiểm tra workflow logs, tìm commit gây ra vấn đề
  4. Tạo hotfix branch, patch và deploy qua pipeline bình thường
  5. Thêm test case cho scenario đó
Result:    Downtime 8 phút. Post-mortem document với action items.
           Cải thiện smoke tests để catch vấn đề trước production.
```

---

### B3: Làm thế nào để onboard một developer mới vào hệ thống CI/CD?

**Điểm cần đề cập:**
- Documentation as code — README.md trong `.github/workflows/`
- Môi trường local testing với `act` tool
- Protected environments — prevent accidental production deploys
- Cặp đôi làm việc với họ trong lần đầu merge PR
- Checklist onboarding có workflow diagrams

---

## Câu Hỏi Bẫy Phổ Biến

### Bẫy 1: "Workflow và Pipeline khác nhau như thế nào?"

Nhiều người dùng hai từ này thay cho nhau, nhưng:
- **Workflow** — file YAML cụ thể trong GitHub Actions
- **Pipeline** — khái niệm rộng hơn, chỉ chuỗi tự động hóa CI/CD

### Bẫy 2: "Có thể dùng GITHUB_TOKEN để push lên repository khác không?"

Không. `GITHUB_TOKEN` chỉ có quyền trên repository đang chạy workflow. Cần PAT hoặc Deploy Key cho cross-repository operations.

### Bẫy 3: "Secrets trong Environment có khác Secrets trong Repository không?"

Có. Environment secrets chỉ accessible khi job chạy trên environment đó (và environment đó có thể yêu cầu approval). Repository secrets accessible từ bất kỳ job nào.

### Bẫy 4: "Cache có được chia sẻ giữa các branches không?"

Có điều kiện:
- Branch có thể dùng cache từ **default branch** (main/master) làm fallback
- Nhưng default branch không dùng cache từ feature branches
- Cache không chia sẻ giữa các repositories

---

## 📊 Tóm Tắt Điểm Quan Trọng

| Chủ Đề | Điểm Cốt Lõi |
|---|---|
| Architecture | Event → Workflow → Job → Step → Action |
| CI Design | Fail fast → test → build → artifact |
| CD Design | Staging → integration tests → manual approval → production |
| OIDC | Short-lived token, no credentials in secrets |
| Reusable | workflow_call = toàn pipeline; composite = nhóm steps |
| Cache | hashFiles() key + restore-keys fallback |
| Security | Pin by SHA, minimal permissions, OIDC > PAT |
| Scale | Central workflows repo, organization secrets, required workflows |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
