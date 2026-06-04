# 1. Caching Dependencies — Bộ Đệm Phụ Thuộc Với `actions/cache`

> `actions/cache` — Action chính thức của GitHub để lưu và khôi phục cache, giúp tái sử dụng dependencies giữa các workflow runs, giảm đáng kể thời gian build.

---

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
2. [Cú Pháp actions/cache](#cú-pháp-actionscache)
3. [Cache cho Node.js / npm / yarn / pnpm](#nodejs--npm--yarn--pnpm)
4. [Cache cho Python / pip / Poetry](#python--pip--poetry)
5. [Cache cho Java / Maven / Gradle](#java--maven--gradle)
6. [Cache cho Go](#go)
7. [Cache cho Rust / Cargo](#rust--cargo)
8. [Cache cho Ruby / Bundler](#ruby--bundler)
9. [Cache tích hợp trong setup actions](#cache-tích-hợp-trong-setup-actions)
10. [Xử Lý Lỗi Cache](#xử-lý-lỗi-cache)

---

## Cơ Chế Hoạt Động

### Luồng Thực Thi

```
Workflow bắt đầu
      │
      ▼
actions/cache (restore)
      │
      ├── Cache HIT ──► Khôi phục cache vào path ──► Bỏ qua install ──► Build/Test
      │
      └── Cache MISS ──► Chạy install bình thường ──► Build/Test
                                                            │
                                                            ▼
                                              (post-job) actions/cache (save)
                                                            │
                                                            ▼
                                                   Lưu cache lên GitHub
```

### Vòng Đời Cache

```
Run #1 (MISS):  npm ci chạy đầy đủ → lưu cache với key "linux-npm-abc123"
Run #2 (HIT):   khôi phục cache "linux-npm-abc123" → npm ci trong ~15 giây
Run #3 (HIT):   package-lock.json chưa đổi → tiếp tục dùng cache
Run #4 (MISS):  package-lock.json đổi → cache key mới → npm ci lại → lưu cache mới
```

---

## Cú Pháp `actions/cache`

```yaml
- uses: actions/cache@v4
  id: cache-deps           # ID để tham chiếu output (cache-hit)
  with:
    path: |                # Đường dẫn cần cache (hỗ trợ nhiều dòng)
      ~/.npm
      node_modules
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |        # Danh sách keys dự phòng (theo thứ tự ưu tiên)
      ${{ runner.os }}-npm-
    save-always: false     # Lưu cache kể cả khi job thất bại (mặc định false)
    enableCrossOsArchive: false  # Cache cross-OS (mặc định false)
```

### Tham Số Quan Trọng

| Tham Số | Bắt Buộc | Mô Tả |
|---|---|---|
| `path` | ✅ | Đường dẫn files/thư mục cần cache |
| `key` | ✅ | Khóa định danh cache, phải duy nhất |
| `restore-keys` | ❌ | Keys dự phòng khi không tìm thấy exact key |
| `save-always` | ❌ | Lưu cache kể cả khi job fail |
| `enableCrossOsArchive` | ❌ | Dùng cache giữa các OS khác nhau |

### Outputs (Đầu Ra)

```yaml
- uses: actions/cache@v4
  id: my-cache
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

- name: Cài đặt nếu cache miss
  if: steps.my-cache.outputs.cache-hit != 'true'
  run: npm ci
```

---

## Node.js / npm / yarn / pnpm

### npm — Cách 1: Cache npm global store (Khuyến Nghị)

```yaml
name: CI Node.js với Cache npm

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'           # setup-node tự xử lý cache (cách đơn giản nhất)

      - name: Cài đặt dependencies
        run: npm ci

      - name: Chạy tests
        run: npm test
```

### npm — Cách 2: Cache thủ công (Kiểm Soát Chi Tiết Hơn)

```yaml
      - name: Cache npm store
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      - name: Cài đặt dependencies
        run: npm ci
```

### yarn v1 — Cache

```yaml
      - name: Lấy thư mục cache của yarn
        id: yarn-cache-dir-path
        run: echo "dir=$(yarn cache dir)" >> $GITHUB_OUTPUT

      - name: Cache yarn dependencies
        uses: actions/cache@v4
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-

      - name: Cài đặt dependencies
        run: yarn install --frozen-lockfile
```

### yarn v2/v3 (Berry) — Cache

```yaml
      - name: Setup Node.js với yarn cache
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'yarn'

      - run: yarn install --immutable
```

### pnpm — Cache

```yaml
      - name: Cài đặt pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: Lấy pnpm store path
        id: pnpm-cache
        run: echo "dir=$(pnpm store path)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-

      - name: Cài đặt dependencies
        run: pnpm install --frozen-lockfile
```

---

## Python / pip / Poetry

### pip — Cache Cơ Bản

```yaml
name: CI Python với Cache pip

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'            # setup-python tự cache pip

      - name: Cài đặt dependencies
        run: pip install -r requirements.txt

      - name: Chạy tests
        run: pytest
```

### pip — Cache Thủ Công

```yaml
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - run: pip install -r requirements.txt
```

### Poetry — Cache Môi Trường Ảo

```yaml
      - name: Cài đặt Poetry
        uses: snok/install-poetry@v1
        with:
          version: 1.8.0
          virtualenvs-create: true
          virtualenvs-in-project: true  # Tạo .venv trong thư mục project

      - name: Load cached venv (Môi Trường Ảo)
        id: cached-poetry-dependencies
        uses: actions/cache@v4
        with:
          path: .venv
          key: venv-${{ runner.os }}-${{ hashFiles('**/poetry.lock') }}

      - name: Cài đặt dependencies (chỉ khi cache miss)
        if: steps.cached-poetry-dependencies.outputs.cache-hit != 'true'
        run: poetry install --no-interaction --no-root

      - name: Cài đặt project
        run: poetry install --no-interaction

      - name: Chạy tests
        run: poetry run pytest
```

### Tox — Cache Nhiều Môi Trường

```yaml
      - name: Cache tox environments
        uses: actions/cache@v4
        with:
          path: .tox
          key: ${{ runner.os }}-tox-${{ hashFiles('tox.ini', 'requirements*.txt') }}

      - run: pip install tox
      - run: tox
```

---

## Java / Maven / Gradle

### Maven — Cache Repository

```yaml
name: CI Java Maven với Cache

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'          # setup-java tự cache Maven local repo

      - name: Build với Maven
        run: mvn --batch-mode --update-snapshots verify
```

### Maven — Cache Thủ Công (Chi Tiết Hơn)

```yaml
      - name: Cache Maven local repository
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Build với Maven
        run: mvn -B package --file pom.xml
```

### Gradle — Cache Caches và Wrapper

```yaml
name: CI Java Gradle với Cache

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'         # setup-java tự cache Gradle

      - name: Build với Gradle
        run: ./gradlew build
```

### Gradle — Cache Thủ Công

```yaml
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Build với Gradle
        run: ./gradlew build
        env:
          GRADLE_OPTS: "-Dorg.gradle.daemon=false"
```

---

## Go

### Go Modules Cache

```yaml
name: CI Go với Cache

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true              # setup-go tự cache Go modules

      - name: Build
        run: go build ./...

      - name: Test
        run: go test ./...
```

### Go — Cache Thủ Công

```yaml
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/go/pkg/mod
            ~/.cache/go-build        # Build cache — tăng tốc compilation
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Tải Go modules
        run: go mod download
```

---

## Rust / Cargo

Rust có thời gian compile rất lâu — cache Cargo giúp tiết kiệm đáng kể.

```yaml
name: CI Rust với Cache

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache Cargo registry và build
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/                    # Thư mục build output — quan trọng nhất
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-

      - name: Build
        run: cargo build --verbose

      - name: Test
        run: cargo test --verbose
```

### Rust — Dùng `Swatinem/rust-cache` (Tối Ưu Hơn)

```yaml
      - name: Cache Rust dependencies
        uses: Swatinem/rust-cache@v2
        with:
          # Cache key tự động dựa vào Cargo.lock và toolchain
          shared-key: "ci"
          save-if: ${{ github.ref == 'refs/heads/main' }}  # Chỉ lưu từ main branch
```

---

## Ruby / Bundler

```yaml
name: CI Ruby với Cache

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true     # setup-ruby tự động cache gems

      - name: Chạy tests
        run: bundle exec rspec
```

### Bundler — Cache Thủ Công

```yaml
      - name: Cache Ruby gems
        uses: actions/cache@v4
        with:
          path: vendor/bundle
          key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-gems-

      - name: Cài đặt gems
        run: |
          bundle config path vendor/bundle
          bundle install --jobs 4 --retry 3
```

---

## Cache Tích Hợp Trong Setup Actions

Nhiều setup actions có tham số `cache` tích hợp sẵn, giúp đơn giản hóa cấu hình:

| Action | Tham Số Cache | Hỗ Trợ |
|---|---|---|
| `actions/setup-node@v4` | `cache: 'npm'` / `'yarn'` / `'pnpm'` | ✅ |
| `actions/setup-python@v5` | `cache: 'pip'` / `'pipenv'` / `'poetry'` | ✅ |
| `actions/setup-java@v4` | `cache: 'maven'` / `'gradle'` / `'sbt'` | ✅ |
| `actions/setup-go@v5` | `cache: true` | ✅ |
| `ruby/setup-ruby@v1` | `bundler-cache: true` | ✅ |
| `actions/setup-dotnet@v4` | Không có cache tích hợp | ❌ |

**Ví dụ tổng hợp:**

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'                    # Tự cache ~/.npm
          cache-dependency-path: |        # Chỉ định lock file cụ thể
            apps/frontend/package-lock.json
            apps/backend/package-lock.json
```

---

## Xử Lý Lỗi Cache

### Kiểm Tra Cache Hit Trước Khi Cài Đặt

```yaml
      - name: Cache dependencies
        id: cache
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

      - name: Cài đặt (chỉ khi cache miss)
        if: steps.cache.outputs.cache-hit != 'true'
        run: npm ci

      - name: Xác minh cài đặt
        run: npm ls --depth=0
```

### Lưu Cache Kể Cả Khi Job Thất Bại

```yaml
      - name: Cache với save-always
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
          save-always: true          # Lưu cache kể cả khi test fail
```

### Debug Cache Issues

```yaml
      - name: Cache (verbose debug)
        uses: actions/cache@v4
        env:
          ACTIONS_CACHE_URL: ${{ env.ACTIONS_CACHE_URL }}
          ACTIONS_RUNTIME_TOKEN: ${{ env.ACTIONS_RUNTIME_TOKEN }}
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-debug-${{ hashFiles('package-lock.json') }}

      - name: Kiểm tra cache
        run: |
          echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
          ls -la ~/.npm | head -20 || echo "Cache directory trống"
```

---

## 📋 Checklist Cache Dependencies

```
✅ Xác định đúng path cần cache (npm store, không phải node_modules)
✅ Sử dụng hashFiles() với lock file tương ứng (package-lock.json, go.sum, ...)
✅ Đặt restore-keys để fallback khi không có exact match
✅ Xem xét dùng cache tích hợp trong setup-* actions
✅ Test workflow với cache miss lần đầu và cache hit lần sau
✅ Kiểm tra cache size không vượt quá 10GB per repo
✅ Không cache files chứa secrets hoặc thông tin nhạy cảm
```

---

## 🔗 Liên Kết Tiếp Theo

- [2-cache-key-strategies.md](./2-cache-key-strategies.md) — Chiến lược khóa nâng cao
- [4-performance-optimization.md](./4-performance-optimization.md) — Tối ưu toàn diện

---

**Cập Nhật Lần Cuối:** 2026-05-12
