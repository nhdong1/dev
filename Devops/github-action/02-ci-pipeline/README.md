# 02 — CI Pipeline (Đường Ống Tích Hợp Liên Tục)

> CI — Continuous Integration (Tích Hợp Liên Tục) — là nền tảng của mọi quy trình DevOps hiện đại. GitHub Actions biến mỗi commit thành một pipeline tự động: kiểm thử, phân tích mã, đóng gói — không cần can thiệp thủ công.

## 📚 Mục Lục Chương

| File | Nội Dung | Độ Quan Trọng |
|---|---|---|
| [1-checkout-setup.md](1-checkout-setup.md) | actions/checkout, setup-node, setup-python, setup-java | ⭐⭐⭐ |
| [2-testing-strategies.md](2-testing-strategies.md) | Unit test, integration test, code coverage, parallelism | ⭐⭐⭐ |
| [3-linting-quality.md](3-linting-quality.md) | ESLint, Prettier, SonarQube, quality gates | ⭐⭐⭐ |
| [4-build-artifacts.md](4-build-artifacts.md) | Build packaging, versioning, upload/download artifacts | ⭐⭐⭐ |
| [5-branch-protection.md](5-branch-protection.md) | Status checks, required reviews, merge rules | ⭐⭐ |

---

## 🏗️ Giải Phẫu Một CI Pipeline Hoàn Chỉnh

```
Developer push code / mở Pull Request
          │
          ▼
  GitHub Actions kích hoạt
          │
    ┌─────▼──────────────────────────────────────┐
    │           CI Pipeline                       │
    │                                             │
    │  Stage 1: Checkout & Setup                  │
    │  ├── actions/checkout@v4                    │
    │  └── actions/setup-node@v4 (hoặc Python...) │
    │                                             │
    │  Stage 2: Install Dependencies              │
    │  └── npm ci / pip install / go mod download │
    │                                             │
    │  Stage 3: Lint & Code Quality               │
    │  ├── ESLint / Prettier                      │
    │  └── SonarQube / CodeQL                     │
    │                                             │
    │  Stage 4: Test                              │
    │  ├── Unit tests                             │
    │  ├── Integration tests                      │
    │  └── Code coverage report                  │
    │                                             │
    │  Stage 5: Build                             │
    │  ├── Compile / Bundle                       │
    │  └── Upload artifacts                       │
    └─────────────────────────────────────────────┘
          │
     Pass ✅ / Fail ❌
          │
    Branch protection kiểm tra status
```

---

## 🔄 Triggers Phổ Biến Cho CI Pipeline

### Kích Hoạt Khi Push Và Pull Request

```yaml
on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
    paths-ignore:           # Bỏ qua thay đổi không ảnh hưởng code
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'

  pull_request:
    branches:
      - main
      - develop
    types:
      - opened            # PR mới được mở
      - synchronize       # Có commit mới vào PR
      - reopened          # PR bị đóng rồi mở lại
```

### Kích Hoạt Thủ Công Để Debug

```yaml
  workflow_dispatch:        # Cho phép chạy thủ công từ GitHub UI
    inputs:
      run_integration_tests:
        description: 'Có chạy integration tests không?'
        type: boolean
        default: false
```

### Merge Queue (Hàng Đợi Merge — GitHub Enterprise)

```yaml
  merge_group:              # Kích hoạt khi PR vào merge queue
    types: [checks_requested]
```

---

## 📋 Template CI Pipeline Hoàn Chỉnh

### Node.js / TypeScript

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
    paths-ignore: ['**.md', 'docs/**']
  pull_request:
    branches: [main, develop]

permissions:
  contents: read            # Quyền tối thiểu — chỉ đọc code

jobs:
  # ─────────────────────────────────────────
  # Job 1: Lint & Type Check
  # ─────────────────────────────────────────
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: TypeScript type check
        run: npm run type-check

  # ─────────────────────────────────────────
  # Job 2: Test
  # ─────────────────────────────────────────
  test:
    name: Test (Node ${{ matrix.node-version }})
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ['18', '20', '22']

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci

      - name: Run unit tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-node-${{ matrix.node-version }}
          path: coverage/
          retention-days: 7

  # ─────────────────────────────────────────
  # Job 3: Build (chỉ sau khi test pass)
  # ─────────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]     # Chờ cả lint và test thành công

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build production bundle
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 30
```

---

## 🐍 Template Python

```yaml
name: CI — Python

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with ruff
        run: ruff check .

      - name: Format check with black
        run: black --check .

      - name: Run tests with coverage
        run: pytest --cov=src --cov-report=xml --cov-fail-under=80

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
```

---

## ☕ Template Java / Maven

```yaml
name: CI — Java

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'       # Eclipse Temurin (OpenJDK)
          cache: 'maven'

      - name: Build and test
        run: mvn -B verify --no-transfer-progress

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()                    # Upload kể cả khi test fail
        with:
          name: test-results
          path: target/surefire-reports/
```

---

## 🔑 Các Khái Niệm Quan Trọng

### needs — Phụ Thuộc Giữa Jobs

```yaml
jobs:
  lint:   ...
  test:   ...
  build:
    needs: [lint, test]   # build chỉ chạy khi cả lint VÀ test pass

  deploy:
    needs: build          # deploy chỉ chạy khi build pass
    if: github.ref == 'refs/heads/main'   # thêm điều kiện nhánh
```

### continue-on-error — Không Fail Toàn Bộ Pipeline

```yaml
steps:
  - name: Run optional analysis
    run: npm run analyze
    continue-on-error: true   # Step fail nhưng job vẫn tiếp tục
```

### timeout-minutes — Giới Hạn Thời Gian

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30       # Job tự hủy sau 30 phút (tránh billing leak)
    steps:
      - name: Long running test
        run: npm run test:e2e
        timeout-minutes: 20   # Từng step cũng có thể đặt timeout
```

### if — Chạy Có Điều Kiện

```yaml
steps:
  - name: Deploy to staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    run: ./deploy.sh staging

  - name: Notify on failure
    if: failure()             # Chỉ chạy khi step trước fail
    uses: slackapi/slack-github-action@v1
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác nhau giữa `push` và `pull_request` trigger?

**Trả lời mẫu:**
- `on: push` kích hoạt khi code được push trực tiếp lên nhánh — thường dùng cho CI trên nhánh `main`/`develop`.
- `on: pull_request` kích hoạt khi PR được mở, cập nhật, hoặc mở lại — thường dùng để validate code **trước** khi merge. Workflow chạy trên merge commit tạm thời (PR head merge với base).

**Điểm phân biệt quan trọng:** `pull_request` từ fork repository có quyền hạn chế hơn (không truy cập secrets) để bảo mật — đây là thiết kế có chủ đích của GitHub.

### Câu 2: Tại sao dùng `npm ci` thay vì `npm install` trong CI?

**Trả lời mẫu:**
- `npm ci` — clean install — xóa `node_modules` hiện có, cài đúng theo `package-lock.json`, không được cập nhật lock file. Đảm bảo build **reproducible** (tái hiện được).
- `npm install` có thể cập nhật `package-lock.json` và cài phiên bản khác — không đảm bảo nhất quán.

**Rule:** Luôn dùng `npm ci` trong CI/CD. `npm install` chỉ dùng khi phát triển local.

### Câu 3: Làm thế nào để chia sẻ data giữa các jobs?

**Trả lời mẫu:**
Có 3 cách:
1. **Artifacts** — upload từ job trước, download ở job sau. Phù hợp cho file lớn (build output, test reports).
2. **Job outputs** — truyền giá trị string nhỏ qua `outputs`. Phù hợp cho version number, flag.
3. **Cache** — dùng `actions/cache` để tái sử dụng dependencies giữa các runs (không phải giữa jobs trong cùng run).

### Câu 4: Khi nào nên dùng `needs:` và khi nào để jobs chạy song song?

**Trả lời mẫu:**
- **Song song (không dùng needs):** Lint + Test có thể chạy song song vì độc lập nhau → tiết kiệm thời gian.
- **Tuần tự (dùng needs):** Build phải sau khi Test pass; Deploy phải sau khi Build xong.

**Best practice:** Tối đa hóa parallelism ở đầu pipeline (lint, test, security scan chạy song song), converge ở cuối (build, deploy chạy tuần tự).

---

## 📂 Điều Hướng

- [→ Checkout & Setup Actions](1-checkout-setup.md)
- [→ Testing Strategies](2-testing-strategies.md)
- [→ Linting & Code Quality](3-linting-quality.md)
- [→ Build & Artifacts](4-build-artifacts.md)
- [→ Branch Protection](5-branch-protection.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
