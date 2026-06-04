# Checkout & Setup Actions — Thiết Lập Môi Trường CI

> Mỗi CI job bắt đầu với môi trường trắng (fresh runner). Ba việc đầu tiên luôn phải làm: lấy code về, cài đúng runtime, tận dụng cache để không mất thời gian tải lại dependencies.

## 📋 Mục Lục

1. [actions/checkout](#actionscheckout)
2. [actions/setup-node](#actionssetup-node)
3. [actions/setup-python](#actionssetup-python)
4. [actions/setup-java](#actionssetup-java)
5. [actions/setup-go](#actionssetup-go)
6. [actions/setup-dotnet](#actionssetup-dotnet)
7. [Sparse Checkout — Checkout Một Phần](#sparse-checkout)
8. [Multi-repo Checkout](#multi-repo-checkout)

---

## actions/checkout

`actions/checkout` — tải source code từ repository vào runner. Đây là step **bắt buộc** đầu tiên trong mọi CI job.

### Cú Pháp Cơ Bản

```yaml
steps:
  - uses: actions/checkout@v4
```

### Tùy Chọn Quan Trọng

```yaml
- uses: actions/checkout@v4
  with:
    # Số lượng commit lấy về (mặc định: 1 — chỉ commit mới nhất)
    # fetch-depth: 0 để lấy toàn bộ history (cần cho git log, changelog)
    fetch-depth: 0

    # Lấy cả tags (cần cho semantic versioning)
    fetch-tags: true

    # Checkout nhánh hoặc tag cụ thể (mặc định: nhánh trigger event)
    ref: 'refs/heads/develop'

    # Token để checkout repo private hoặc tạo commit
    token: ${{ secrets.GITHUB_TOKEN }}

    # Checkout vào thư mục con (hữu ích khi checkout nhiều repo)
    path: 'my-app'

    # Tắt submodules (mặc định: false)
    submodules: recursive
```

### Khi Nào Dùng fetch-depth: 0

```yaml
# Cần full history để:
# 1. Tính toán git log / changelog
# 2. Chạy git blame / git bisect
# 3. Công cụ như commitizen, semantic-release, conventional-commits
# 4. SonarQube phân tích blame information

- uses: actions/checkout@v4
  with:
    fetch-depth: 0          # Lấy toàn bộ history
```

```yaml
# Chỉ cần commit gần nhất (mặc định — nhanh hơn):
- uses: actions/checkout@v4
  with:
    fetch-depth: 1          # Hoặc bỏ qua, mặc định là 1
```

### Tạo Commit Từ CI (Write-back)

```yaml
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.PAT_TOKEN }}   # PAT — Personal Access Token — cần để push
                                       # GITHUB_TOKEN không trigger workflow mới

- name: Update version file
  run: echo "1.2.3" > VERSION

- name: Commit and push
  run: |
    git config user.name  "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add VERSION
    git commit -m "chore: bump version to 1.2.3"
    git push
```

---

## actions/setup-node

Cài đặt Node.js — Node Package Manager trên runner và tùy chọn cache `node_modules`.

### Cú Pháp Đầy Đủ

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'          # Phiên bản cụ thể
    # node-version: '20.x'      # Minor version mới nhất của 20
    # node-version: 'lts/*'     # LTS — Long Term Support — mới nhất
    # node-version-file: '.nvmrc'  # Đọc từ file .nvmrc hoặc .node-version

    cache: 'npm'                # Cache npm registry downloads
    # cache: 'yarn'
    # cache: 'pnpm'

    cache-dependency-path: 'package-lock.json'  # Mặc định, có thể tùy chỉnh
    # cache-dependency-path: '**/package-lock.json'  # Monorepo
```

### Đọc Phiên Bản Từ File

```yaml
# package.json có "engines": { "node": ">=20" }
- uses: actions/setup-node@v4
  with:
    node-version-file: 'package.json'

# .nvmrc chứa "20.11.0"
- uses: actions/setup-node@v4
  with:
    node-version-file: '.nvmrc'
```

### Pattern Đầy Đủ Với Cache

```yaml
- uses: actions/checkout@v4

- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

- name: Install dependencies
  run: npm ci               # Cài từ package-lock.json, reproducible

- name: Build
  run: npm run build

- name: Test
  run: npm test
```

### pnpm / yarn

```yaml
# pnpm
- uses: pnpm/action-setup@v4
  with:
    version: 9

- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'

- run: pnpm install --frozen-lockfile

# yarn
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'yarn'

- run: yarn install --immutable
```

---

## actions/setup-python

Cài đặt Python và tùy chọn cache pip packages.

### Cú Pháp Đầy Đủ

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'        # Phiên bản cụ thể
    # python-version: '3.x'       # Phiên bản 3.x mới nhất
    # python-version-file: '.python-version'  # Đọc từ file

    cache: 'pip'                  # Cache pip downloads
    # cache: 'pipenv'
    # cache: 'poetry'

    cache-dependency-path: 'requirements*.txt'  # Glob pattern
```

### Pattern Với pip

```yaml
- uses: actions/checkout@v4

- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
    cache-dependency-path: |
      requirements.txt
      requirements-dev.txt

- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
    pip install -r requirements-dev.txt

- name: Run tests
  run: pytest --cov=src --cov-report=xml
```

### Pattern Với Poetry

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'poetry'

- name: Install Poetry
  run: pip install poetry

- name: Install dependencies
  run: poetry install --no-root

- name: Run tests
  run: poetry run pytest
```

### Ma Trận Nhiều Phiên Bản Python

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - run: pip install -r requirements-dev.txt
      - run: pytest
```

---

## actions/setup-java

Cài đặt Java JDK — Java Development Kit và build tools Maven / Gradle.

### Cú Pháp Đầy Đủ

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'       # Bản phân phối JDK
    # distribution: 'corretto'    # Amazon Corretto
    # distribution: 'microsoft'   # Microsoft Build of OpenJDK
    # distribution: 'zulu'        # Azul Zulu
    # distribution: 'oracle'      # Oracle JDK (cần license cho production)

    cache: 'maven'                # Cache Maven local repository
    # cache: 'gradle'

    # Cài đặt Maven settings.xml cho private registry
    server-id: 'github'
    server-username: ${{ github.actor }}
    server-password: ${{ secrets.GITHUB_TOKEN }}
```

### Cú Pháp Đọc Từ File

```yaml
# .java-version chứa "21"
- uses: actions/setup-java@v4
  with:
    java-version-file: '.java-version'
    distribution: 'temurin'
    cache: 'maven'
```

### Pattern Maven

```yaml
- uses: actions/checkout@v4

- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'

- name: Build and test
  run: mvn -B verify --no-transfer-progress

- name: Upload JAR artifact
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: target/*.jar
```

### Pattern Gradle

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'gradle'

- name: Grant execute permission
  run: chmod +x gradlew

- name: Build and test
  run: ./gradlew build

- name: Test report
  uses: actions/upload-artifact@v4
  with:
    name: gradle-test-results
    path: build/reports/tests/
```

---

## actions/setup-go

Cài đặt Go và cache Go modules.

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'              # Phiên bản cụ thể
    # go-version: '1.x'             # Mới nhất của 1.x
    # go-version-file: 'go.mod'     # Đọc từ go.mod

    cache: true                     # Cache Go module cache (mặc định: true)
    cache-dependency-path: go.sum   # File lock

- name: Download dependencies
  run: go mod download

- name: Run tests
  run: go test ./... -coverprofile=coverage.out

- name: Build
  run: go build -v ./...
```

---

## actions/setup-dotnet

Cài đặt .NET SDK.

```yaml
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '8.x'           # Phiên bản .NET 8 mới nhất
    # dotnet-version: |             # Cài đặt nhiều phiên bản
    #   6.x
    #   8.x

- name: Restore dependencies
  run: dotnet restore

- name: Build
  run: dotnet build --no-restore

- name: Test
  run: dotnet test --no-build --verbosity normal
```

---

## Sparse Checkout

Sparse Checkout (Checkout Thưa Thớt) — chỉ tải về phần code cần thiết, không tải toàn bộ repo. Hữu ích với monorepo (kho mã đơn — một repo chứa nhiều project).

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      packages/api
      packages/shared
      package.json
    sparse-checkout-cone-mode: true   # Cone mode — nhanh hơn, dùng prefix matching
```

```yaml
# Ví dụ monorepo: chỉ build service "api" khi code trong packages/api thay đổi
on:
  push:
    paths:
      - 'packages/api/**'
      - 'packages/shared/**'

jobs:
  build-api:
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            packages/api
            packages/shared
```

---

## Multi-repo Checkout

Checkout nhiều repository vào cùng một job.

```yaml
steps:
  # Checkout repo chính vào thư mục mặc định
  - uses: actions/checkout@v4
    with:
      path: main-repo

  # Checkout repo phụ (shared library)
  - uses: actions/checkout@v4
    with:
      repository: my-org/shared-utils
      token: ${{ secrets.PAT_TOKEN }}   # Cần PAT nếu repo private
      path: shared-utils
      ref: 'v2.0.0'

  - name: Use both repos
    run: |
      ls main-repo/
      ls shared-utils/
      # Chạy script sử dụng code từ cả hai repo
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Tại sao nên pin actions theo SHA thay vì tag?

**Trả lời mẫu:**
Tag như `actions/checkout@v4` có thể bị tác giả trỏ lại (re-tag) sang commit khác — đây là rủi ro supply chain attack (tấn công chuỗi cung ứng). SHA như `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` là bất biến — không ai có thể thay đổi nội dung đằng sau SHA đó.

**Thực tế:** Dùng Dependabot để tự động cập nhật SHA khi có phiên bản mới, giữ được tính bảo mật mà không mất tiện lợi.

### Câu 2: Sự khác nhau giữa `cache: 'npm'` trong setup-node và `actions/cache`?

**Trả lời mẫu:**
- `cache: 'npm'` trong `actions/setup-node` là shortcut — tự động cache thư mục npm global cache (`~/.npm`). Đơn giản, ít config.
- `actions/cache` là action thủ công — cache bất kỳ thư mục nào, custom key, restore keys phức tạp. Dùng khi cần kiểm soát chi tiết hơn (ví dụ: cache `node_modules` thay vì npm global cache).

### Câu 3: fetch-depth: 0 ảnh hưởng gì đến performance?

**Trả lời mẫu:**
Với repo lớn có nhiều năm history, `fetch-depth: 0` có thể tốn thêm 30–60 giây để clone toàn bộ. `fetch-depth: 1` (mặc định) chỉ lấy commit mới nhất — nhanh hơn đáng kể. Chỉ dùng `fetch-depth: 0` khi thực sự cần: semantic release, git-based changelogs, hoặc phân tích history.

---

## 📂 Điều Hướng

- [← Quay lại README](README.md)
- [→ Testing Strategies](2-testing-strategies.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
