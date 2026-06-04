# Build & Artifacts — Đóng Gói và Lưu Trữ Kết Quả Build

> Artifacts (Tệp Đầu Ra) là cầu nối giữa CI và CD. Build job tạo ra binary / image / package; artifact lưu lại kết quả đó để deployment job tiêu thụ — không cần build lại, đảm bảo "build once, deploy many" (xây một lần, triển khai nhiều lần).

## 📋 Mục Lục

1. [Build Fundamentals](#build-fundamentals)
2. [actions/upload-artifact](#actionsupload-artifact)
3. [actions/download-artifact](#actionsdownload-artifact)
4. [Versioning — Đánh Số Phiên Bản](#versioning)
5. [Multi-platform Builds](#multi-platform-builds)
6. [Docker Image Build](#docker-image-build)
7. [Publishing Packages](#publishing-packages)
8. [Build Matrix và Artifact Management](#build-matrix-và-artifact-management)

---

## Build Fundamentals

### "Build Once, Deploy Many" Pattern

```
CI Pipeline:
  build job → upload artifact (build-${{ github.sha }})
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
 deploy-dev    deploy-staging  deploy-prod
  ↳ download   ↳ download     ↳ download
    artifact     artifact       artifact
    (same!)      (same!)        (same!)

Lợi ích:
- Không build lại → tiết kiệm thời gian và cost
- Binary giống hệt nhau ở mọi env → loại bỏ "works on staging, fails on prod"
- Audit trail: biết chính xác artifact nào đang chạy ở đâu
```

---

## actions/upload-artifact

### Cú Pháp Đầy Đủ

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: my-artifact              # Tên duy nhất trong workflow run
    path: dist/                    # File hoặc thư mục cần upload

    # Thời gian giữ lại (mặc định: 90 ngày, max: 90 ngày free / 400 ngày paid)
    retention-days: 30

    # Hành vi khi artifact đã tồn tại
    overwrite: false               # false: fail nếu trùng tên (mặc định)
    # overwrite: true              # Ghi đè nếu trùng tên

    # Không upload nếu không tìm thấy file (thay vì fail)
    if-no-files-found: error       # error (mặc định) | warn | ignore

    # Nén files (mặc định: zip)
    compression-level: 6           # 0 (không nén) đến 9 (nén tối đa)
```

### Pattern Upload Build Output

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Build production bundle
        run: npm run build
        env:
          NODE_ENV: production

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: web-build-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map          # Loại trừ source maps (dùng ! prefix)
          retention-days: 30
          if-no-files-found: error
```

### Upload Nhiều Artifacts

```yaml
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()              # Upload kể cả khi test fail
        with:
          name: test-results
          path: test-results/
          retention-days: 7

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

      - name: Upload build logs
        uses: actions/upload-artifact@v4
        if: failure()             # Chỉ upload khi có lỗi để debug
        with:
          name: build-logs
          path: logs/
          retention-days: 3
```

---

## actions/download-artifact

### Tải Artifact Từ Cùng Workflow Run

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist               # Phải khớp với tên đã upload
          path: dist/              # Thư mục đích (tự tạo nếu chưa có)

      - name: Deploy
        run: |
          ls dist/
          ./deploy.sh dist/
```

### Tải Nhiều Artifacts

```yaml
      # Tải một artifact cụ thể
      - uses: actions/download-artifact@v4
        with:
          name: web-build
          path: artifacts/web

      # Tải tất cả artifacts (không chỉ định name)
      - uses: actions/download-artifact@v4
        with:
          path: all-artifacts      # Mỗi artifact trong thư mục con riêng
          # Kết quả: all-artifacts/artifact-1/, all-artifacts/artifact-2/, ...

      # Tải artifacts theo pattern
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-*      # Tải tất cả artifact bắt đầu bằng "coverage-"
          path: coverage-reports
          merge-multiple: true     # Merge tất cả vào cùng thư mục
```

### Tải Artifact Từ Workflow Run Khác

```yaml
      - name: Download artifact from another run
        uses: dawidd6/action-download-artifact@v6
        with:
          workflow: build.yml
          branch: main
          name: production-build
          path: downloaded/
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Versioning

Semantic Versioning — SemVer (Đánh Số Phiên Bản Ngữ Nghĩa): `MAJOR.MINOR.PATCH` (ví dụ: `2.1.3`).

### Git SHA Làm Version (Đơn Giản Nhất)

```yaml
      - name: Build with SHA version
        run: |
          VERSION=${{ github.sha }}
          SHORT_SHA=${VERSION::7}         # 7 ký tự đầu của SHA
          echo "Building version: ${SHORT_SHA}"
          docker build -t my-app:${SHORT_SHA} .

      - name: Tag with branch+SHA
        run: |
          BRANCH=$(echo ${{ github.ref_name }} | tr '/' '-')
          TAG="${BRANCH}-${{ github.run_number }}"
          echo "VERSION=${TAG}" >> $GITHUB_ENV
```

### Semantic Release — Tự Động Tăng Version

`semantic-release` đọc conventional commit messages và tự động:
- Tăng version (feat→minor, fix→patch, BREAKING→major)
- Tạo CHANGELOG
- Tạo GitHub Release
- Publish package

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: write               # Cần để tạo tag và release
      issues: write
      pull-requests: write
      id-token: write               # Cần cho npm provenance

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # Full history để semantic-release tính version
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

### Manual Version Tag

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., 1.2.3)'
        required: true
        type: string

jobs:
  tag-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.PAT_TOKEN }}

      - name: Create and push tag
        run: |
          git tag "v${{ inputs.version }}"
          git push origin "v${{ inputs.version }}"

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: "v${{ inputs.version }}"
          generate_release_notes: true    # Auto-generate từ merged PRs
```

### Đọc Version Từ package.json / pom.xml

```yaml
      - name: Get version from package.json
        id: version
        run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT

      - name: Use version
        run: echo "Building version ${{ steps.version.outputs.version }}"
```

```yaml
      - name: Get Maven version
        id: version
        run: echo "version=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)" >> $GITHUB_OUTPUT
```

---

## Multi-platform Builds

### Ma Trận OS

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        include:
          - os: ubuntu-latest
            artifact-name: app-linux
            binary-ext: ''
          - os: windows-latest
            artifact-name: app-windows
            binary-ext: '.exe'
          - os: macos-latest
            artifact-name: app-macos
            binary-ext: ''

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Build binary
        run: go build -o app${{ matrix.binary-ext }} ./cmd/app

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.artifact-name }}
          path: app${{ matrix.binary-ext }}

  # Tổng hợp tất cả binaries vào một release
  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: binaries/
          pattern: app-*
          merge-multiple: false

      - name: Create release with all binaries
        uses: softprops/action-gh-release@v2
        with:
          files: |
            binaries/app-linux/app
            binaries/app-windows/app.exe
            binaries/app-macos/app
```

---

## Docker Image Build

### Build và Push Lên Registry

```yaml
jobs:
  docker-build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write               # Cần để push lên GHCR — GitHub Container Registry

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        # Buildx — BuildKit extension — hỗ trợ multi-platform và cache mạnh hơn

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64     # Multi-arch build
          push: ${{ github.event_name != 'pull_request' }}  # Không push cho PR
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha                   # Dùng GitHub Actions cache
          cache-to: type=gha,mode=max
```

### Build Docker Và Lưu Như Artifact (Không Push)

```yaml
      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .

      - name: Export Docker image as tar
        run: docker save my-app:${{ github.sha }} | gzip > my-app.tar.gz

      - name: Upload image artifact
        uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: my-app.tar.gz
          retention-days: 1         # Chỉ cần trong cùng pipeline run
```

```yaml
  # Job sau tải về và load image
  deploy:
    needs: docker-build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: docker-image

      - name: Load Docker image
        run: docker load < my-app.tar.gz

      - name: Deploy
        run: docker run -d my-app:${{ github.sha }}
```

---

## Publishing Packages

### npm Publish

```yaml
jobs:
  publish-npm:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')    # Chỉ publish khi có tag version
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
      - run: npm run build
      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### GitHub Packages (npm registry trên GitHub)

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'
          scope: '@my-org'

      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### PyPI Publish

```yaml
jobs:
  publish-pypi:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    environment: pypi                   # Dùng GitHub Environment để kiểm soát approval
    permissions:
      id-token: write                   # OIDC để authenticate với PyPI (không cần API key)

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Build package
        run: |
          pip install build
          python -m build

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        # Dùng OIDC — không cần PYPI_API_TOKEN
```

---

## Build Matrix và Artifact Management

### Collect Artifacts Từ Matrix Jobs

```yaml
jobs:
  build:
    strategy:
      matrix:
        platform: [linux, windows, macos]
    runs-on: ${{ matrix.platform == 'windows' && 'windows-latest' || matrix.platform == 'macos' && 'macos-latest' || 'ubuntu-latest' }}

    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: make build
      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.platform }}
          path: build/

  package:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download all platform binaries
        uses: actions/download-artifact@v4
        with:
          pattern: binary-*
          path: all-binaries/
          merge-multiple: false         # Giữ cấu trúc thư mục riêng

      - name: List downloaded artifacts
        run: find all-binaries/ -type f

      - name: Create release archive
        run: |
          cd all-binaries
          for dir in */; do
            platform="${dir%/}"
            zip -r "../${platform}.zip" "${dir}"
          done

      - name: Upload combined release
        uses: actions/upload-artifact@v4
        with:
          name: release-packages
          path: '*.zip'
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Tại sao "build once, deploy many" quan trọng?

**Trả lời mẫu:**
Nếu mỗi environment build lại từ đầu, có rủi ro:
1. Dependencies thay đổi giữa các builds (npm registry update package mới)
2. Build environment khác nhau tạo ra binary khác nhau
3. Tốn thêm thời gian và cost

Build once, deploy many đảm bảo binary đã test trên staging là chính xác binary được deploy lên production — không có sự khác biệt.

### Câu 2: Retention policy cho artifacts nên đặt bao nhiêu?

**Trả lời mẫu:**
Phụ thuộc vào mục đích:
- **Test reports, coverage:** 7–14 ngày (debug PR)
- **Build artifacts cho deployment:** 30 ngày (rollback nếu cần)
- **Release artifacts:** Lưu lên GitHub Releases hoặc S3 vĩnh viễn (không dùng workflow artifacts)
- **Debug logs:** 3–7 ngày (chỉ cần khi debug gần đây)

Artifact quá lâu tốn storage cost. GitHub miễn phí 500MB/tháng cho public repos.

### Câu 3: Semantic versioning MAJOR.MINOR.PATCH — khi nào tăng cái nào?

**Trả lời mẫu:**
- **PATCH** (`1.0.0` → `1.0.1`): Bug fix, không thay đổi API
- **MINOR** (`1.0.0` → `1.1.0`): Thêm tính năng mới, backward-compatible (không phá vỡ API cũ)
- **MAJOR** (`1.0.0` → `2.0.0`): Breaking change (thay đổi phá vỡ API — code cũ dùng thư viện này sẽ bị lỗi)

Với `semantic-release`: `fix:` → patch, `feat:` → minor, `BREAKING CHANGE:` trong footer → major.

---

## 📂 Điều Hướng

- [← Linting & Code Quality](3-linting-quality.md)
- [→ Branch Protection](5-branch-protection.md)
- [↑ Quay lại README](README.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
