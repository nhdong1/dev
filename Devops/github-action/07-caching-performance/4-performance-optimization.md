# 4. Performance Optimization — Tối Ưu Thời Gian Chạy Workflow

> Workflow nhanh = developer feedback nhanh = năng suất cao hơn. Bài này trình bày toàn diện các kỹ thuật giảm thời gian chạy GitHub Actions workflows.

---

## 📚 Mục Lục

1. [Đo Lường Trước Khi Tối Ưu](#đo-lường-trước-khi-tối-ưu)
2. [Parallel Jobs — Chạy Song Song](#parallel-jobs)
3. [Conditional Steps — Bỏ Qua Bước Không Cần Thiết](#conditional-steps)
4. [Path Filters — Chỉ Chạy Khi Cần](#path-filters)
5. [Shallow Clone — Clone Nhanh Hơn](#shallow-clone)
6. [Dependency Caching — Tái Sử Dụng Dependencies](#dependency-caching)
7. [Build Caching — Cache Kết Quả Build](#build-caching)
8. [Job Concurrency — Kiểm Soát Đồng Thời](#job-concurrency)
9. [Skip CI — Bỏ Qua CI Khi Không Cần](#skip-ci)
10. [Tối Ưu Docker Builds](#tối-ưu-docker-builds)
11. [Tối Ưu Test Execution](#tối-ưu-test-execution)
12. [Self-Hosted Runners Cho Tốc Độ](#self-hosted-runners-cho-tốc-độ)

---

## Đo Lường Trước Khi Tối Ưu

### Xem Thời Gian Từng Step

Mỗi step trong GitHub Actions UI hiển thị thời gian chạy. Nhìn vào:
- Steps nào chiếm nhiều thời gian nhất?
- Có jobs nào đang chờ nhau không cần thiết?
- Cache hit rate là bao nhiêu?

### Tính Toán Hiện Trạng

```yaml
# Thêm step đo lường thời gian
- name: Bắt đầu đo lường
  run: echo "START_TIME=$(date +%s)" >> $GITHUB_ENV

# ... các bước khác ...

- name: Kết thúc đo lường
  run: |
    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))
    echo "Tổng thời gian: ${DURATION} giây"
```

### Workflow Timing Summary

```bash
# Dùng GitHub CLI để xem timing của workflow runs gần đây
gh run list --workflow=ci.yml --limit=10 \
  --json databaseId,status,conclusion,createdAt,updatedAt \
  | jq '.[] | {id: .databaseId, duration: (.updatedAt | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime) - (.createdAt | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime)}'
```

---

## Parallel Jobs — Chạy Song Song

### Trước Khi Tối Ưu (Sequential — Tuần Tự)

```yaml
# ❌ Chậm — các jobs chạy tuần tự
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint        # 2 phút

  test:
    needs: lint                  # Chờ lint xong mới chạy
    runs-on: ubuntu-latest
    steps:
      - run: npm test            # 3 phút

  build:
    needs: test                  # Chờ test xong mới chạy
    runs-on: ubuntu-latest
    steps:
      - run: npm run build       # 2 phút

# Tổng: 7 phút
```

### Sau Khi Tối Ưu (Parallel — Song Song)

```yaml
# ✅ Nhanh — lint, test chạy song song; build chờ cả hai
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint        # 2 phút

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test            # 3 phút

  build:
    needs: [lint, test]         # Chờ cả lint VÀ test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build       # 2 phút

# Tổng: max(2, 3) + 2 = 5 phút (tiết kiệm 2 phút = 28%)
```

### Thiết Kế Dependency Graph Tối Ưu

```
❌ Sequential:   lint → test → security-scan → build → deploy
                 2m     3m         4m            2m      1m = 12m

✅ Parallel:
   ┌─ lint ──────────────────────────────────┐
   ├─ test ──────────────────────────────────┤
   └─ security-scan ─────────────────────────┘ → build → deploy
   max(2, 3, 4) = 4m                            2m        1m = 7m
```

---

## Conditional Steps — Bỏ Qua Bước Không Cần Thiết

### Dùng `if` Để Bỏ Qua Steps

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Chỉ chạy linting trên PRs, không phải mỗi push lên main
      - name: Lint code
        if: github.event_name == 'pull_request'
        run: npm run lint

      # Chỉ build docker image trên main branch
      - name: Build Docker image
        if: github.ref == 'refs/heads/main'
        run: docker build -t myapp .

      # Chỉ deploy khi không phải dependabot PR
      - name: Deploy
        if: |
          github.ref == 'refs/heads/main' &&
          github.actor != 'dependabot[bot]'
        run: ./deploy.sh
```

### Conditional Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'        # Chỉ deploy từ main
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy-staging.sh

  deploy-production:
    needs: deploy-staging
    if: startsWith(github.ref, 'refs/tags/v')   # Chỉ deploy từ version tags
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: ./deploy-production.sh
```

### Detect Changed Files Để Chạy Có Chọn Lọc

```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend-changed: ${{ steps.filter.outputs.frontend }}
      backend-changed: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            frontend:
              - 'apps/frontend/**'
              - 'packages/ui/**'
            backend:
              - 'apps/backend/**'
              - 'packages/shared/**'

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:frontend    # Chỉ chạy nếu frontend thay đổi

  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:backend     # Chỉ chạy nếu backend thay đổi
```

---

## Path Filters — Chỉ Chạy Khi Cần

### Trigger Theo Đường Dẫn

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'           # Chỉ trigger khi source code thay đổi
      - 'package*.json'
      - '.github/workflows/**'
    paths-ignore:
      - '**.md'            # Bỏ qua thay đổi markdown
      - 'docs/**'
      - '.gitignore'

  pull_request:
    paths:
      - 'src/**'
      - 'tests/**'
```

### Ví Dụ Thực Tế: Monorepo Triggers

```yaml
# ci-frontend.yml — chỉ khi frontend thay đổi
on:
  push:
    paths:
      - 'apps/frontend/**'
      - 'packages/ui/**'
      - '.github/workflows/ci-frontend.yml'

# ci-backend.yml — chỉ khi backend thay đổi
on:
  push:
    paths:
      - 'apps/backend/**'
      - 'packages/shared/**'
      - '.github/workflows/ci-backend.yml'

# ci-infra.yml — chỉ khi infrastructure thay đổi
on:
  push:
    paths:
      - 'terraform/**'
      - 'kubernetes/**'
      - '.github/workflows/ci-infra.yml'
```

---

## Shallow Clone — Clone Nhanh Hơn

### Vấn Đề Với Clone Đầy Đủ

Repository lớn với lịch sử dài → clone chậm:
```
git clone → download 500MB lịch sử commit → 30–60 giây
```

### Giải Pháp: Shallow Clone

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1        # Chỉ lấy commit mới nhất (mặc định: 1)
          # fetch-depth: 0      # Lịch sử đầy đủ (cần cho git log, semantic-release)
```

### Khi Nào Cần `fetch-depth: 0`

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Cần toàn bộ lịch sử cho:
                            # - git log để tạo CHANGELOG
                            # - semantic-release để tính version
                            # - GitVersion hoặc công cụ similar
                            # - Code coverage comparison với base branch
```

### Clone Chỉ Một Branch

```yaml
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}   # Clone branch hiện tại
          fetch-depth: 1
          # Không fetch các branches khác → nhanh hơn
```

---

## Dependency Caching — Tái Sử Dụng Dependencies

(Xem chi tiết tại [1-caching-dependencies.md](./1-caching-dependencies.md))

### Quick Reference

```yaml
# npm
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'              # Cách đơn giản nhất

# Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'

# Java Maven
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'

# Go
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true
```

---

## Build Caching — Cache Kết Quả Build

### TypeScript Incremental Build Cache

```yaml
      - name: Cache TypeScript build
        uses: actions/cache@v4
        with:
          path: .tsbuildinfo     # TypeScript incremental build info
          key: ${{ runner.os }}-tsc-${{ hashFiles('src/**/*.ts', 'tsconfig.json') }}

      - name: Build TypeScript
        run: npx tsc --build    # Chỉ compile files đã thay đổi
```

### Next.js Build Cache

```yaml
      - name: Cache Next.js build
        uses: actions/cache@v4
        with:
          path: |
            ~/.npm
            ${{ github.workspace }}/.next/cache    # Next.js build cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**.[jt]s', '**.[jt]sx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-
            ${{ runner.os }}-nextjs-

      - run: npm run build
```

### Gradle Build Cache

```yaml
      - name: Cache Gradle build output
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
            build/
          key: ${{ runner.os }}-gradle-build-${{ hashFiles('**/*.gradle*', 'src/**/*.java') }}

      - name: Build với Gradle
        run: ./gradlew build --build-cache    # Bật Gradle build cache
```

---

## Job Concurrency — Kiểm Soát Đồng Thời

### Tránh Chạy Nhiều Workflow Cùng Lúc

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true      # Hủy run cũ khi có run mới hơn

# Tác dụng:
# Developer push commit 1 → CI bắt đầu
# Developer push commit 2 → CI commit 1 bị hủy → CI commit 2 chạy
# Tiết kiệm minutes, feedback nhanh hơn
```

### Concurrency Theo Môi Trường Deploy

```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-${{ github.ref }}-production
      cancel-in-progress: false   # Không hủy deployment đang chạy (nguy hiểm)
```

### Concurrency Cho Pull Requests

```yaml
concurrency:
  # PR: mỗi PR có group riêng, hủy run cũ
  group: ci-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

---

## Skip CI — Bỏ Qua CI Khi Không Cần

### Bằng Commit Message

```yaml
on:
  push:
    branches: [main]

jobs:
  ci:
    # Bỏ qua nếu commit message chứa [skip ci] hoặc [ci skip]
    if: |
      !contains(github.event.head_commit.message, '[skip ci]') &&
      !contains(github.event.head_commit.message, '[ci skip]')
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

**Dùng trong commit message:**
```bash
git commit -m "docs: cập nhật README [skip ci]"
git commit -m "[ci skip] fix: typo trong comment"
```

### GitHub Built-in Skip

GitHub tự động bỏ qua CI khi commit message chứa:
- `[skip ci]`
- `[ci skip]`
- `[no ci]`
- `[skip actions]`
- `[actions skip]`

---

## Tối Ưu Docker Builds

### Layer Caching (Caching Theo Lớp)

```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build và Push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: myapp:${{ github.sha }}
          cache-from: type=gha           # Dùng GitHub Actions cache
          cache-to: type=gha,mode=max    # Lưu tất cả layers vào cache
```

### Multi-Stage Build Optimization

```dockerfile
# Stage 1: Dependencies (ít thay đổi → cache lâu dài)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production    # Chỉ production deps

# Stage 2: Build (thay đổi thường xuyên hơn)
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Runtime (image nhỏ nhất có thể)
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

### Buildx với Registry Cache

```yaml
      - uses: docker/build-push-action@v6
        with:
          context: .
          tags: ${{ env.REGISTRY }}/myapp:latest
          # Dùng registry để cache layers giữa các machines
          cache-from: type=registry,ref=${{ env.REGISTRY }}/myapp:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/myapp:buildcache,mode=max
```

---

## Tối Ưu Test Execution

### Chạy Tests Song Song

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5]    # Chia thành 5 shards song song
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      # Jest
      - run: npx jest --shard=${{ matrix.shard }}/5 --ci

      # Playwright
      - run: npx playwright test --shard=${{ matrix.shard }}/5
```

### Chỉ Chạy Tests Bị Ảnh Hưởng (Jest)

```yaml
      # Chỉ chạy tests liên quan đến files đã thay đổi
      - name: Chạy tests bị ảnh hưởng
        run: |
          npx jest \
            --onlyChanged \             # Chỉ test files đã thay đổi
            --passWithNoTests \         # Không fail nếu không có test nào
            --ci \
            --coverage
```

### Test Order Optimization (Sắp Xếp Tests Theo Thứ Tự)

```yaml
      # Chạy tests nhanh trước, tests chậm sau → fail fast
      - name: Unit tests (nhanh)
        run: npm run test:unit          # ~30 giây

      - name: Integration tests (vừa)
        run: npm run test:integration   # ~2 phút

      - name: E2E tests (chậm nhất)
        run: npm run test:e2e           # ~5 phút
```

---

## Self-Hosted Runners Cho Tốc Độ

### Khi Nào Dùng Self-Hosted Runners

| Tình Huống | GitHub-hosted | Self-hosted |
|---|---|---|
| Projects nhỏ/vừa | ✅ | Không cần |
| Build time > 10 phút | Tốn kém | ✅ Tiết kiệm |
| Cần hardware đặc biệt (GPU, ARM) | ❌ | ✅ |
| Cần truy cập private network | ❌ | ✅ |
| Cần cache lớn (>10GB) | ❌ | ✅ |
| Cần tốc độ network cao | Giới hạn | ✅ |

### Self-Hosted Runner Với Cache Local

```yaml
# Workflow sử dụng local cache (không tốn bandwidth GitHub)
jobs:
  build:
    runs-on: self-hosted    # Runner có SSD nhanh, cache sẵn sàng
    steps:
      - uses: actions/checkout@v4

      # Cache trên disk của runner (nhanh hơn nhiều so với GitHub cache)
      - name: Cache dependencies (local)
        uses: actions/cache@v4
        with:
          path: /opt/runner-cache/npm    # Path trên disk runner
          key: npm-${{ hashFiles('package-lock.json') }}
```

---

## 📊 Tóm Tắt Kỹ Thuật Tối Ưu

| Kỹ Thuật | Tiết Kiệm Điển Hình | Độ Phức Tạp | Áp Dụng Ngay |
|---|---|---|---|
| Dependency caching | 50–80% install time | Thấp | ✅ |
| Parallel jobs | 30–60% total time | Trung bình | ✅ |
| Path filters | 60–90% triggers | Thấp | ✅ |
| Shallow clone | 10–30% checkout time | Rất thấp | ✅ |
| Concurrency groups | Giảm queuing time | Thấp | ✅ |
| Skip CI | 100% (khi không cần) | Rất thấp | ✅ |
| Test sharding | 50–80% test time | Trung bình | Cân nhắc |
| Docker layer cache | 50–70% build time | Trung bình | Cân nhắc |
| Self-hosted runners | 40–70% total time | Cao | Dự án lớn |
| Build caching (tsc, gradle) | 30–50% build time | Trung bình | Cân nhắc |

---

## ✅ Checklist Tối Ưu Workflow

```
Cơ bản (Làm ngay — < 30 phút):
✅ Bật dependency cache (setup-node cache: 'npm')
✅ Dùng fetch-depth: 1 cho checkout
✅ Thêm path filters để trigger chọn lọc
✅ Thêm concurrency group để hủy runs cũ

Trung cấp (Đáng đầu tư — 1–2 giờ):
✅ Tách lint/test/build thành parallel jobs
✅ Dùng dorny/paths-filter cho monorepo
✅ Cache Docker layers với type=gha
✅ Thêm conditional jobs/steps

Nâng cao (Khi cần thiết — nhiều giờ):
✅ Sharding tests với matrix strategy
✅ Self-hosted runners cho workloads lớn
✅ Remote Turborepo/Nx caching
✅ Build caching cho TypeScript/Gradle
```

---

## 🔗 Liên Kết Tiếp Theo

- [5-billing-cost.md](./5-billing-cost.md) — Tính toán ROI khi tối ưu

---

**Cập Nhật Lần Cuối:** 2026-05-12
