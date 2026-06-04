# 3. Artifacts — Tệp Đầu Ra: Upload, Download và Retention Policy

> Artifacts (Tệp Đầu Ra) là cơ chế lưu trữ và chia sẻ files giữa các jobs trong cùng một workflow, hoặc lưu kết quả build/test để download sau khi workflow hoàn thành.

---

## 📚 Mục Lục

1. [Artifacts vs Cache — Khi Nào Dùng Cái Nào?](#artifacts-vs-cache)
2. [Upload Artifacts — Tải Lên Tệp Đầu Ra](#upload-artifacts)
3. [Download Artifacts — Tải Xuống Tệp Đầu Ra](#download-artifacts)
4. [Retention Policy — Chính Sách Lưu Trữ](#retention-policy)
5. [Chia Sẻ Files Giữa Jobs](#chia-sẻ-files-giữa-jobs)
6. [Test Reports — Báo Cáo Kiểm Thử](#test-reports)
7. [Build Artifacts — Tệp Build](#build-artifacts)
8. [Quản Lý Artifacts Qua API](#quản-lý-artifacts-qua-api)
9. [Best Practices](#best-practices)

---

## Artifacts vs Cache

| Tiêu Chí | Artifacts (Tệp Đầu Ra) | Cache (Bộ Đệm) |
|---|---|---|
| **Mục đích chính** | Lưu kết quả build, test reports, release binaries | Tái sử dụng dependencies giữa runs |
| **Phạm vi chia sẻ** | Giữa jobs trong cùng run, download thủ công | Giữa nhiều workflow runs |
| **Thời hạn** | 1–400 ngày (mặc định 90) | 7 ngày không được dùng |
| **Giới hạn storage** | Theo plan (mỗi artifact tối đa 2GB) | 10GB per repo |
| **Tự động xóa** | Sau retention period | Sau 7 ngày không active |
| **Download** | Từ UI hoặc API | Chỉ trong workflow |

**Dùng Artifacts khi:**
- Truyền build output từ job `build` sang job `deploy`
- Lưu test reports để xem sau
- Publish release binaries để download
- Lưu logs/screenshots khi test fail

**Dùng Cache khi:**
- Tăng tốc `npm install`, `pip install`, `mvn build`
- Tái sử dụng compiled dependencies

---

## Upload Artifacts — Tải Lên Tệp Đầu Ra

### Cú Pháp Cơ Bản

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: my-artifact            # Tên artifact (bắt buộc)
    path: dist/                  # Đường dẫn file/thư mục (bắt buộc)
    retention-days: 30           # Giữ trong 30 ngày (mặc định: 90)
    if-no-files-found: error     # Xử lý khi không tìm thấy file: error/warn/ignore
    compression-level: 6         # Mức nén 0-9 (mặc định: 6)
    overwrite: false             # Ghi đè nếu tên đã tồn tại (mặc định: false)
```

### Upload Một File

```yaml
      - name: Build ứng dụng
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
```

### Upload Nhiều Files

```yaml
      # Cách 1: Upload nhiều paths trong một artifact
      - uses: actions/upload-artifact@v4
        with:
          name: combined-output
          path: |
            dist/
            coverage/
            reports/*.xml

      # Cách 2: Upload nhiều artifacts riêng lẻ
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

      - uses: actions/upload-artifact@v4
        with:
          name: test-coverage
          path: coverage/
          retention-days: 7      # Test coverage giữ ngắn hơn
```

### Upload Với Wildcard (Ký Tự Đại Diện)

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            **/test-results.xml
            **/junit.xml
            !node_modules/**      # Loại trừ node_modules
```

### Upload File Nén (Khi Size Quan Trọng)

```yaml
      - name: Đóng gói trước khi upload
        run: tar -czf release.tar.gz dist/ public/

      - uses: actions/upload-artifact@v4
        with:
          name: release-package
          path: release.tar.gz
          compression-level: 0   # File đã nén rồi, không cần nén lại
```

### Upload Kể Cả Khi Job Thất Bại

```yaml
      - name: Chạy tests
        run: npm test
        continue-on-error: true

      - name: Upload test results (luôn upload)
        uses: actions/upload-artifact@v4
        if: always()             # Chạy kể cả khi step trước fail
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

---

## Download Artifacts — Tải Xuống Tệp Đầu Ra

### Download Trong Cùng Workflow Run

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    needs: build              # Chờ job build hoàn thành
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output   # Tên phải khớp với upload
          path: dist/          # Đường dẫn đích (mặc định: thư mục hiện tại)

      - name: Deploy
        run: |
          ls -la dist/
          # Thực hiện deploy...
```

### Download Tất Cả Artifacts

```yaml
      - uses: actions/download-artifact@v4
        # Không chỉ định "name" → download tất cả artifacts
        # Mỗi artifact sẽ được đặt trong thư mục con cùng tên

      - name: Liệt kê tất cả artifacts
        run: find . -type f | head -50
```

### Download Từ Workflow Run Khác

```yaml
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          run-id: ${{ github.event.workflow_run.id }}  # ID của run khác
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Download Artifact Từ Branch Khác (Workflow Trigger)

```yaml
name: Deploy từ Build

on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          path: ./dist
```

---

## Retention Policy — Chính Sách Lưu Trữ

### Cài Đặt Mặc Định

| Cấp Độ | Cách Cài Đặt | Phạm Vi |
|---|---|---|
| Repository settings | Settings → Actions → Artifact and log retention | Tất cả workflows trong repo |
| Organization settings | Org settings → Actions → Artifact retention | Tất cả repos trong org |
| Per-artifact | `retention-days` trong workflow | Artifact cụ thể |

**Lưu ý:** `retention-days` trong workflow không thể vượt quá cài đặt repository/org.

### Cài Đặt Per-Artifact

```yaml
      # Test results — giữ ngắn, chỉ cần debug gần đây
      - uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/
          retention-days: 7

      # Release binaries — giữ lâu hơn
      - uses: actions/upload-artifact@v4
        with:
          name: release-v${{ github.ref_name }}
          path: dist/
          retention-days: 365

      # Debug logs — giữ rất ngắn
      - uses: actions/upload-artifact@v4
        with:
          name: debug-logs
          path: logs/
          retention-days: 3
```

### Chiến Lược Retention Theo Loại Artifact

```
Type                  | Retention | Lý Do
──────────────────────|───────────|────────────────────────────────
Test reports          | 7–14 ngày | Chỉ cần debug issues gần đây
Code coverage         | 7–30 ngày | So sánh với runs gần đây
Build output (dev)    | 7–14 ngày | Tạm thời để kiểm tra
Build output (prod)   | 90–365 ngày| Có thể rollback
Release binaries      | 365 ngày  | Download bởi users
Security scan results | 90 ngày   | Compliance, audit
```

---

## Chia Sẻ Files Giữa Jobs

### Pattern Cơ Bản: Build → Test → Deploy

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - uses: actions/checkout@v4

      - name: Đặt phiên bản
        id: version
        run: echo "value=$(cat VERSION)" >> $GITHUB_OUTPUT

      - name: Build
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ steps.version.outputs.value }}
          path: dist/
          retention-days: 7

  test-e2e:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: build-${{ needs.build.outputs.version }}
          path: dist/

      - name: Chạy E2E tests
        run: npm run test:e2e

  deploy-staging:
    needs: [build, test-e2e]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-${{ needs.build.outputs.version }}
          path: dist/

      - name: Deploy lên Staging
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
          # deploy commands...
```

### Pattern: Fan-Out Testing (Kiểm Thử Song Song)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]    # Chia tests thành 4 phần chạy song song
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Chạy test shard ${{ matrix.shard }}
        run: npm run test -- --shard=${{ matrix.shard }}/4

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-shard-${{ matrix.shard }}
          path: test-results/

  merge-results:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: test-results-shard-*
          merge-multiple: true    # Gộp tất cả vào thư mục hiện tại
          path: all-results/

      - name: Tổng hợp kết quả
        run: |
          ls all-results/
          # Merge XML reports, tính tổng coverage, ...
```

---

## Test Reports — Báo Cáo Kiểm Thử

### JUnit XML Reports

```yaml
      - name: Chạy tests
        run: |
          pytest tests/ \
            --junit-xml=test-results/junit.xml \
            --cov=src \
            --cov-report=xml:coverage/coverage.xml
        continue-on-error: true

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            test-results/junit.xml
            coverage/coverage.xml
          retention-days: 14
```

### Screenshots và Videos Khi E2E Tests Fail

```yaml
      - name: Chạy Playwright E2E tests
        run: npx playwright test
        continue-on-error: true

      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: |
            playwright-report/
            test-results/          # Chứa screenshots và videos
          retention-days: 7
```

### Coverage Reports

```yaml
      - name: Chạy tests với coverage
        run: npm run test:coverage

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 30

      # Tùy chọn: publish coverage lên GitHub Pages
      - name: Deploy coverage lên GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: coverage/lcov-report
```

---

## Build Artifacts — Tệp Build

### Release Artifacts (Tệp Phát Hành)

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build ứng dụng
        run: |
          npm run build:prod
          tar -czf app-${{ github.ref_name }}.tar.gz dist/

      - name: Upload release artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-${{ github.ref_name }}
          path: app-${{ github.ref_name }}.tar.gz
          retention-days: 365    # Giữ release artifacts lâu dài

  create-release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: release-${{ github.ref_name }}

      - name: Tạo GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: app-${{ github.ref_name }}.tar.gz
```

### Docker Image Tar (Không Dùng Registry)

```yaml
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker save myapp:${{ github.sha }} | gzip > myapp-image.tar.gz

      - uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: myapp-image.tar.gz
          retention-days: 1       # Chỉ cần cho workflow run này

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: docker-image

      - name: Load Docker image
        run: docker load < myapp-image.tar.gz

      - name: Deploy
        run: docker run -d myapp:${{ github.sha }}
```

---

## Quản Lý Artifacts Qua API

### Liệt Kê Artifacts

```bash
# Liệt kê artifacts của một workflow run
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/owner/repo/actions/runs/$RUN_ID/artifacts"

# Dùng GitHub CLI
gh api repos/owner/repo/actions/runs/$RUN_ID/artifacts
```

### Xóa Artifact

```bash
# Xóa artifact theo ID
curl -X DELETE \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/owner/repo/actions/artifacts/$ARTIFACT_ID"
```

### Download Artifact Từ Workflow

```bash
# Download artifact qua GitHub CLI
gh run download $RUN_ID --name build-output --dir ./downloads/

# Download tất cả artifacts
gh run download $RUN_ID --dir ./all-artifacts/
```

### Tự Động Xóa Artifacts Cũ

```yaml
name: Cleanup Artifacts (Dọn Dẹp Artifacts Cũ)

on:
  schedule:
    - cron: '0 0 * * 0'    # Chạy mỗi Chủ Nhật lúc 00:00 UTC

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Xóa artifacts cũ hơn 30 ngày
        uses: actions/github-script@v7
        with:
          script: |
            const thirtyDaysAgo = new Date();
            thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

            const { data: { artifacts } } = await github.rest.actions.listArtifactsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              per_page: 100,
            });

            for (const artifact of artifacts) {
              if (new Date(artifact.created_at) < thirtyDaysAgo) {
                await github.rest.actions.deleteArtifact({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  artifact_id: artifact.id,
                });
                console.log(`Đã xóa artifact: ${artifact.name} (${artifact.id})`);
              }
            }
```

---

## Best Practices

### Đặt Tên Artifact Rõ Ràng

```yaml
# KHÔNG NÊN — tên chung chung, khó phân biệt
name: output

# NÊN — tên mô tả đầy đủ
name: build-linux-amd64-${{ github.ref_name }}
name: test-results-unit-${{ github.run_id }}
name: coverage-report-pr-${{ github.event.number }}
```

### Luôn Dùng `if: always()` Cho Test Artifacts

```yaml
      - name: Chạy tests
        run: npm test
        continue-on-error: true

      # Phải có if: always() nếu không artifact sẽ không được upload khi test fail
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results/
```

### Tránh Upload Files Quá Lớn

```yaml
      # Kiểm tra size trước khi upload
      - name: Kiểm tra build size
        run: |
          SIZE=$(du -sm dist/ | cut -f1)
          echo "Build size: ${SIZE}MB"
          if [ $SIZE -gt 500 ]; then
            echo "⚠️ Build quá lớn (${SIZE}MB > 500MB)"
          fi

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          compression-level: 9   # Nén tối đa cho files lớn
```

### Retention Phù Hợp Theo Môi Trường

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: ${{ github.ref == 'refs/heads/main' && 30 || 7 }}
          # main branch: 30 ngày | PR branches: 7 ngày
```

---

## 📋 Tóm Tắt

```
Upload artifacts  → actions/upload-artifact@v4
Download artifacts → actions/download-artifact@v4

Quy tắc vàng:
✅ Upload test reports với if: always() để có dù fail
✅ Đặt retention-days phù hợp với từng loại artifact
✅ Đặt tên artifact mô tả, bao gồm version hoặc run ID
✅ Dùng artifacts để truyền build output giữa jobs
✅ Không cache secrets trong artifacts
✅ Kiểm soát size — tối đa 2GB per artifact
```

---

## 🔗 Liên Kết Tiếp Theo

- [4-performance-optimization.md](./4-performance-optimization.md) — Tối ưu thời gian chạy
- [5-billing-cost.md](./5-billing-cost.md) — Quản lý chi phí storage

---

**Cập Nhật Lần Cuối:** 2026-05-12
