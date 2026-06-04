# Fan-out & Fan-in — Phân Tán và Tập Hợp Kết Quả

> Fan-out (Phân Tán) chia công việc thành nhiều jobs song song; Fan-in (Tập Hợp) gom kết quả lại từ tất cả nhánh song song. Kết hợp hai pattern này giúp rút ngắn thời gian pipeline đáng kể.

---

## 📚 Mục Lục

1. [Khái Niệm Fan-out và Fan-in](#1-khái-niệm-fan-out-và-fan-in)
2. [Fan-out Cơ Bản — Phân Tán Công Việc](#2-fan-out-cơ-bản--phân-tán-công-việc)
3. [Fan-in — Tập Hợp Kết Quả](#3-fan-in--tập-hợp-kết-quả)
4. [Truyền Dữ Liệu Giữa Jobs](#4-truyền-dữ-liệu-giữa-jobs)
5. [Sharded Testing — Test Phân Mảnh](#5-sharded-testing--test-phân-mảnh)
6. [Dynamic Fan-out — Phân Tán Động](#6-dynamic-fan-out--phân-tán-động)
7. [Multi-level Fan-out/Fan-in](#7-multi-level-fan-outfan-in)
8. [Thực Hành: Các Kịch Bản Thực Tế](#8-thực-hành-các-kịch-bản-thực-tế)
9. [Anti-Patterns](#9-anti-patterns)
10. [Tóm Tắt](#10-tóm-tắt)

---

## 1. Khái Niệm Fan-out và Fan-in

### Định Nghĩa

```
                    ┌─ Job A ─┐
Trigger ──► [Fan-out] ─ Job B ─ [Fan-in] ──► Kết Quả Cuối
                    └─ Job C ─┘

Fan-out (Phân Tán): Chia một luồng thành nhiều nhánh song song
Fan-in  (Tập Hợp): Gom nhiều nhánh về một điểm tập hợp
```

### Lợi Ích

| Chạy Tuần Tự | Chạy Fan-out/Fan-in |
|---|---|
| Test A: 5 phút | Test A: 5 phút ┐ |
| Test B: 5 phút | Test B: 5 phút ├─ song song → 5 phút |
| Test C: 5 phút | Test C: 5 phút ┘ |
| **Tổng: 15 phút** | **Tổng: 5 phút + overhead** |

### Nguồn Gốc Thuật Ngữ

- **Fan-out**: Trong điện tử, fan-out là số tín hiệu đầu ra từ một cổng — nghĩa là "phân nhánh ra nhiều hướng"
- **Fan-in**: Ngược lại — "gom về một điểm"
- Trong CI/CD: fan-out = parallel jobs, fan-in = aggregation job dùng `needs`

---

## 2. Fan-out Cơ Bản — Phân Tán Công Việc

### Cách 1: Nhiều Jobs Độc Lập

```yaml
name: Parallel Testing

on: push

jobs:
  # Cả ba jobs chạy ĐỒNG THỜI vì không có `needs`
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration

  test-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:e2e
```

### Cách 2: Fan-out Bằng Matrix Strategy

```yaml
jobs:
  test:
    strategy:
      matrix:
        suite: [unit, integration, e2e, performance]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:${{ matrix.suite }}
```

**Khi nào dùng cách nào?**

| Cách | Phù Hợp Khi |
|---|---|
| Jobs riêng biệt | Logic mỗi job khác nhau nhiều |
| Matrix | Logic giống nhau, chỉ khác tham số |

---

## 3. Fan-in — Tập Hợp Kết Quả

### `needs` — Chờ Nhiều Jobs

```yaml
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  test-e2e:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e

  # Fan-in: job này chỉ chạy khi cả 3 jobs trên đều hoàn thành
  deploy:
    needs: [test-unit, test-integration, test-e2e]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Fan-in Với `if: always()` — Xem Kết Quả Dù Có Lỗi

```yaml
  report:
    needs: [test-unit, test-integration, test-e2e]
    runs-on: ubuntu-latest
    # Luôn chạy, kể cả khi một trong các jobs thất bại
    if: always()
    steps:
      - name: Check results
        run: |
          echo "unit:        ${{ needs.test-unit.result }}"
          echo "integration: ${{ needs.test-integration.result }}"
          echo "e2e:         ${{ needs.test-e2e.result }}"

      - name: Fail if any test failed
        if: |
          needs.test-unit.result == 'failure' ||
          needs.test-integration.result == 'failure' ||
          needs.test-e2e.result == 'failure'
        run: exit 1
```

### Kiểm Tra Kết Quả Từ Matrix Job

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        suite: [unit, integration, e2e]
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:${{ matrix.suite }}

  # Fan-in từ matrix job
  all-tests-passed:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      # needs.test.result = 'success' chỉ khi TẤT CẢ matrix jobs thành công
      - name: Fail if any matrix job failed
        if: needs.test.result != 'success'
        run: |
          echo "Một hoặc nhiều test suites thất bại!"
          exit 1
```

---

## 4. Truyền Dữ Liệu Giữa Jobs

### Job Outputs (Đầu Ra Job)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      # Khai báo outputs sẽ truyền cho jobs tiếp theo
      image-tag: ${{ steps.build.outputs.tag }}
      build-time: ${{ steps.build.outputs.time }}
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        id: build
        run: |
          TAG="$GITHUB_SHA"
          docker build -t myapp:$TAG .
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          echo "time=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_OUTPUT

  # Fan-in: nhận outputs từ build job
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying image tag: ${{ needs.build.outputs.image-tag }}"
          ./deploy.sh ${{ needs.build.outputs.image-tag }}

  deploy-canary:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Canary deploy
        run: ./deploy-canary.sh ${{ needs.build.outputs.image-tag }}
```

### Truyền Dữ Liệu Lớn Bằng Artifacts

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --json > test-results.json
      - name: Upload test results
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.json

  report:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Download test results
        uses: actions/download-artifact@v4
        with:
          name: test-results
      - name: Generate report
        run: node generate-report.js test-results.json
```

### Fan-in Với Dữ Liệu Từ Nhiều Matrix Jobs

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        with:
          # Mỗi shard upload artifact tên riêng
          name: coverage-shard-${{ matrix.shard }}
          path: coverage/

  merge-coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      # Download tất cả artifacts từ tất cả shards
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-shard-*
          merge-multiple: true
          path: coverage/

      - name: Merge coverage reports
        run: npx nyc merge coverage merged-coverage.json

      - name: Generate HTML report
        run: npx nyc report --reporter=html
```

---

## 5. Sharded Testing — Test Phân Mảnh

### Khái Niệm Sharding (Phân Mảnh)

**Sharding** chia test suite lớn thành các phần nhỏ (shards — mảnh), mỗi shard chạy trên một runner riêng song song. Đây là kỹ thuật phổ biến để rút ngắn thời gian CI cho dự án lớn.

```
Test Suite: 1000 tests (30 phút nếu chạy tuần tự)

Sharding (4 shards):
  Shard 1/4: tests 1–250   → 7.5 phút
  Shard 2/4: tests 251–500 → 7.5 phút
  Shard 3/4: tests 501–750 → 7.5 phút
  Shard 4/4: tests 751–1000→ 7.5 phút
  → Tổng: ~7.5 phút (thay vì 30)
```

### Ví Dụ: Jest Sharding

```yaml
name: Sharded Jest Tests

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4, 5]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Run shard ${{ matrix.shard }}/5
        run: npx jest --shard=${{ matrix.shard }}/5 --ci --coverage

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: jest-results-shard-${{ matrix.shard }}
          path: |
            coverage/
            test-results.xml

  merge-results:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci

      - name: Download all shard results
        uses: actions/download-artifact@v4
        with:
          pattern: jest-results-shard-*
          merge-multiple: true
          path: all-results/

      - name: Merge coverage
        run: |
          npx nyc merge all-results/coverage merged-coverage.json
          npx nyc report --reporter=lcov --reporter=html

      - name: Check all shards passed
        if: needs.test.result != 'success'
        run: |
          echo "❌ Một hoặc nhiều shards thất bại"
          exit 1
```

### Ví Dụ: Playwright E2E Sharding

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard-index: [1, 2, 3]
        shard-total: [3]   # Tổng số shards (dùng trong command)

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium

      - name: Run Playwright shard ${{ matrix.shard-index }}/${{ matrix.shard-total }}
        run: |
          npx playwright test \
            --shard=${{ matrix.shard-index }}/${{ matrix.shard-total }} \
            --reporter=blob

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: blob-report-${{ matrix.shard-index }}
          path: blob-report/
          retention-days: 1

  merge-playwright-reports:
    needs: e2e
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci

      - uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports/

      - name: Merge Playwright reports
        run: |
          npx playwright merge-reports \
            --reporter=html \
            ./all-blob-reports

      - uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14
```

---

## 6. Dynamic Fan-out — Phân Tán Động

### Vấn Đề

Số lượng "nhánh" fan-out không biết trước — ví dụ, danh sách microservices thay đổi theo thời gian.

### Giải Pháp: Job Sinh Matrix Động

```yaml
name: Build Changed Services

on:
  push:
    branches: [main]

jobs:
  # Bước 1: Tìm services bị thay đổi
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
      has-changes: ${{ steps.detect.outputs.has-changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2   # Cần lịch sử để diff

      - name: Detect changed services
        id: detect
        run: |
          # Tìm services có file thay đổi
          CHANGED=$(git diff --name-only HEAD~1 HEAD \
            | grep -E '^services/[^/]+/' \
            | sed 's|services/\([^/]*\)/.*|\1|' \
            | sort -u)

          if [ -z "$CHANGED" ]; then
            echo "has-changes=false" >> $GITHUB_OUTPUT
            echo "services=[]" >> $GITHUB_OUTPUT
          else
            # Chuyển thành JSON array
            SERVICES=$(echo "$CHANGED" | jq -R . | jq -s -c .)
            echo "has-changes=true" >> $GITHUB_OUTPUT
            echo "services=$SERVICES" >> $GITHUB_OUTPUT
          fi

  # Bước 2: Fan-out — Build song song các services bị thay đổi
  build-services:
    needs: detect-changes
    if: needs.detect-changes.outputs.has-changes == 'true'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}

    steps:
      - uses: actions/checkout@v4
      - name: Build ${{ matrix.service }}
        run: |
          docker build \
            -t myregistry.io/myapp/${{ matrix.service }}:${{ github.sha }} \
            services/${{ matrix.service }}/
      - name: Push image
        run: docker push myregistry.io/myapp/${{ matrix.service }}:${{ github.sha }}

  # Bước 3: Fan-in — Deploy tất cả sau khi build xong
  deploy-all:
    needs: [detect-changes, build-services]
    if: needs.build-services.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy all changed services
        run: |
          SERVICES='${{ needs.detect-changes.outputs.services }}'
          echo "$SERVICES" | jq -r '.[]' | while read service; do
            kubectl set image deployment/$service \
              $service=myregistry.io/myapp/$service:${{ github.sha }}
          done
```

---

## 7. Multi-level Fan-out/Fan-in

### Kiến Trúc Hai Tầng

```
                 ┌─ build-arm64 ─┐
build-base ──► ─┤               ├─ push-manifest ──► deploy
                 └─ build-amd64 ─┘
```

```yaml
name: Multi-arch Build and Deploy

jobs:
  # Tầng 0: Build base image chung
  build-base:
    runs-on: ubuntu-latest
    outputs:
      base-tag: ${{ steps.tag.outputs.value }}
    steps:
      - uses: actions/checkout@v4
      - id: tag
        run: echo "value=base-${{ github.sha }}" >> $GITHUB_OUTPUT
      - run: docker build -t myapp:${{ steps.tag.outputs.value }}-base --target base .

  # Tầng 1: Fan-out — Build song song nhiều architectures
  build-arch:
    needs: build-base
    strategy:
      matrix:
        arch: [amd64, arm64]
        include:
          - arch: amd64
            runner: ubuntu-latest
            platform: linux/amd64
          - arch: arm64
            runner: ubuntu-24.04-arm
            platform: linux/arm64
    runs-on: ${{ matrix.runner }}
    outputs:
      # Output cho mỗi arch (indexed by matrix)
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build ${{ matrix.arch }}
        run: |
          docker build \
            --platform ${{ matrix.platform }} \
            --build-arg BASE_TAG=${{ needs.build-base.outputs.base-tag }} \
            -t myregistry.io/myapp:${{ github.sha }}-${{ matrix.arch }} .
      - name: Push
        id: push
        run: |
          docker push myregistry.io/myapp:${{ github.sha }}-${{ matrix.arch }}
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' \
            myregistry.io/myapp:${{ github.sha }}-${{ matrix.arch }})
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT

  # Tầng 2: Fan-in — Tạo multi-arch manifest
  create-manifest:
    needs: build-arch
    runs-on: ubuntu-latest
    steps:
      - name: Create and push manifest
        run: |
          docker manifest create myregistry.io/myapp:${{ github.sha }} \
            myregistry.io/myapp:${{ github.sha }}-amd64 \
            myregistry.io/myapp:${{ github.sha }}-arm64
          docker manifest push myregistry.io/myapp:${{ github.sha }}

  # Tầng 3: Deploy sau khi manifest sẵn sàng
  deploy:
    needs: create-manifest
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: kubectl set image deployment/myapp myapp=myregistry.io/myapp:${{ github.sha }}
```

---

## 8. Thực Hành: Các Kịch Bản Thực Tế

### Kịch Bản 1: Monorepo CI/CD

```yaml
name: Monorepo CI/CD

on:
  push:
    branches: [main]

jobs:
  # 1. Detect thay đổi
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend:  ${{ steps.filter.outputs.backend }}
      infra:    ${{ steps.filter.outputs.infra }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            frontend:
              - 'frontend/**'
            backend:
              - 'backend/**'
            infra:
              - 'infra/**'

  # 2. Fan-out: Build các phần có thay đổi (song song)
  build-frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd frontend && npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: frontend/dist/

  build-backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd backend && go build ./...
      - uses: actions/upload-artifact@v4
        with:
          name: backend-binary
          path: backend/bin/

  # 3. Fan-in: Deploy tất cả khi builds xong
  deploy:
    needs: [changes, build-frontend, build-backend]
    # Chạy nếu ít nhất một trong hai build jobs thành công
    if: |
      always() && (
        needs.build-frontend.result == 'success' ||
        needs.build-backend.result == 'success'
      )
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4

      - name: Deploy frontend
        if: needs.build-frontend.result == 'success'
        run: aws s3 sync frontend-dist/ s3://my-bucket/ --delete

      - name: Deploy backend
        if: needs.build-backend.result == 'success'
        run: kubectl set image deployment/backend backend=myimage:${{ github.sha }}
```

### Kịch Bản 2: Security Scan Fan-out

```yaml
name: Security Checks

on:
  pull_request:

jobs:
  # Fan-out: tất cả scan chạy song song
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run CodeQL
        uses: github/codeql-action/analyze@v3

  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t scan-target .
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target
          exit-code: '1'
          severity: CRITICAL,HIGH

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2

  # Fan-in: Gate — block merge nếu bất kỳ scan nào fail
  security-gate:
    needs: [sast, dependency-check, container-scan, secret-scan]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Evaluate security gate
        run: |
          RESULTS=(
            "${{ needs.sast.result }}"
            "${{ needs.dependency-check.result }}"
            "${{ needs.container-scan.result }}"
            "${{ needs.secret-scan.result }}"
          )
          FAILED=0
          for result in "${RESULTS[@]}"; do
            if [[ "$result" == "failure" ]]; then
              FAILED=$((FAILED + 1))
            fi
          done
          if [[ $FAILED -gt 0 ]]; then
            echo "❌ $FAILED security check(s) failed. Cannot merge!"
            exit 1
          fi
          echo "✅ All security checks passed"
```

---

## 9. Anti-Patterns

### ❌ Fan-in Không Có `if: always()`

```yaml
# Sai: Nếu một job fail, report job bị skip (không chạy)
# → Không biết tại sao pipeline fail
report:
  needs: [test-a, test-b, test-c]
  runs-on: ubuntu-latest
  # Không có if: always() → bị skip khi có job fail
  steps:
    - run: ./generate-report.sh
```

```yaml
# Đúng: Luôn tạo report để có thể debug
report:
  needs: [test-a, test-b, test-c]
  runs-on: ubuntu-latest
  if: always()
  steps:
    - run: ./generate-report.sh
    - name: Fail if tests failed
      if: needs.test-a.result != 'success' || needs.test-b.result != 'success'
      run: exit 1
```

### ❌ Fan-out Quá Nhiều Jobs Nhỏ

```yaml
# Sai: Mỗi test file là một job → overhead startup cao hơn thời gian test
strategy:
  matrix:
    test-file:
      - tests/auth.test.js       # 30 giây test
      - tests/users.test.js      # 20 giây test
      - tests/orders.test.js     # 25 giây test
      # Runner startup: ~30-60 giây → overhead lớn hơn test thực tế
```

```yaml
# Đúng: Nhóm tests thành shards hợp lý (mỗi shard vài phút)
strategy:
  matrix:
    shard: [1, 2, 3]
# Mỗi shard chạy nhiều tests → amortize (phân bổ) startup cost
```

### ❌ Không Xử Lý Artifact Từ Matrix Jobs

```yaml
# Sai: Nhiều matrix jobs upload cùng tên artifact → ghi đè nhau
strategy:
  matrix:
    shard: [1, 2, 3]
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: test-results   # Tên giống nhau cho tất cả shards!
      path: results/
```

```yaml
# Đúng: Tên artifact kèm shard index
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: test-results-shard-${{ matrix.shard }}
      path: results/
```

---

## 10. Tóm Tắt

### Khi Nào Dùng Fan-out/Fan-in?

| Tình Huống | Fan-out Pattern |
|---|---|
| Test suite lớn (>5 phút) | Sharded testing |
| Test đa môi trường/OS | Matrix strategy |
| Build đa nền tảng | Jobs riêng theo platform |
| Scan bảo mật song song | Multiple security jobs |
| Deploy đa cloud | Matrix với cloud providers |
| Monorepo, nhiều package | Conditional jobs theo changes |

### Template Fan-out/Fan-in Cơ Bản

```yaml
jobs:
  # === Fan-out ===
  job-a:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.work.outputs.result }}
    steps:
      - id: work
        run: echo "result=done-a" >> $GITHUB_OUTPUT

  job-b:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.work.outputs.result }}
    steps:
      - id: work
        run: echo "result=done-b" >> $GITHUB_OUTPUT

  # === Fan-in ===
  aggregate:
    needs: [job-a, job-b]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Aggregate results
        run: |
          echo "A: ${{ needs.job-a.outputs.result }}"
          echo "B: ${{ needs.job-b.outputs.result }}"
      - name: Gate
        if: needs.job-a.result != 'success' || needs.job-b.result != 'success'
        run: exit 1
```

### Câu Hỏi Phỏng Vấn

**Q: Fan-out và fan-in pattern là gì trong GitHub Actions?**

> Fan-out là chia công việc thành nhiều jobs song song (dùng matrix strategy hoặc nhiều jobs độc lập). Fan-in là gom kết quả lại bằng `needs` — job fan-in chỉ chạy khi tất cả jobs trước hoàn thành. Thường dùng cho sharded testing, multi-platform builds, parallel security scans.

**Q: Cách truyền dữ liệu từ fan-out jobs về fan-in job?**

> Hai cách: (1) Job outputs — phù hợp với dữ liệu nhỏ (string, JSON). (2) Artifacts — phù hợp với files lớn như test reports, coverage, binaries. Fan-in job download artifacts với `pattern:` để lấy tất cả từ matrix jobs.

**Q: Sharding tests khác gì với matrix strategy?**

> Thực ra sharding dùng matrix strategy — chúng là cùng một cơ chế. Sharding là pattern cụ thể: chia test suite thành N phần bằng nhau, mỗi phần chạy trên một matrix instance. Khác biệt là: matrix strategy có thể dùng cho mọi loại fan-out, còn sharding chuyên cho bài toán chia đều test cases.

---

**Cập Nhật:** 2026-05-11 | **Quay Lại:** [README.md](README.md)
