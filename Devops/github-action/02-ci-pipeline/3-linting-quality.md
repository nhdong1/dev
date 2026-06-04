# Linting & Code Quality — Phân Tích Chất Lượng Mã

> Linting — phân tích mã tĩnh — phát hiện lỗi trước khi test chạy. Quality gates (ngưỡng chất lượng) ngăn code tệ merge vào main. Tự động hóa những việc này trong CI giải phóng reviewer để focus vào logic, không phải style.

## 📋 Mục Lục

1. [Tổng Quan Linting Pipeline](#tổng-quan-linting-pipeline)
2. [ESLint — JavaScript / TypeScript](#eslint)
3. [Prettier — Code Formatter](#prettier)
4. [Python Linters](#python-linters)
5. [Go / Java / Other Languages](#go--java--other-languages)
6. [SonarQube / SonarCloud](#sonarqube--sonarcloud)
7. [Super-linter — All-in-one](#super-linter)
8. [Quality Gates và Thresholds](#quality-gates-và-thresholds)
9. [Auto-fix và Auto-commit](#auto-fix-và-auto-commit)

---

## Tổng Quan Linting Pipeline

```
Code thay đổi (push / PR)
        │
        ▼
┌───────────────────────────────────────────┐
│  Lint Stage (chạy song song với Test)     │
│                                           │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Formatter│  │  Linter  │  │ Type    │ │
│  │ (Prettier│  │ (ESLint) │  │ Checker │ │
│  │  Black)  │  │          │  │ (tsc)   │ │
│  └──────────┘  └──────────┘  └─────────┘ │
│                                           │
│  ┌──────────────────────────────────────┐ │
│  │  SAST — Static Application Security  │ │
│  │  Testing (Kiểm Tra Bảo Mật Tĩnh)    │ │
│  │  SonarQube / CodeQL / Semgrep        │ │
│  └──────────────────────────────────────┘ │
└───────────────────────────────────────────┘
        │
   Pass ✅ → Merge được
   Fail ❌ → Block merge, annotate PR
```

---

## ESLint

ESLint — công cụ phân tích mã tĩnh (static analysis) phổ biến nhất cho JavaScript và TypeScript.

### Setup Cơ Bản

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Run ESLint
        run: npx eslint . --ext .js,.ts,.tsx --max-warnings 0
        # --max-warnings 0: bất kỳ warning nào cũng fail (strict mode)
```

### ESLint Với Annotations (Gắn Chú Thích Trên PR)

```yaml
      - name: Run ESLint with annotations
        uses: reviewdog/action-eslint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review   # Đăng comment trực tiếp trên PR diff
          eslint_flags: '--ext .js,.ts,.tsx'
          fail_on_error: true
```

### ESLint Output Formats

```yaml
      - name: Run ESLint (JSON output)
        run: npx eslint . --format json --output-file eslint-report.json
        continue-on-error: true

      - name: Annotate ESLint results
        uses: ataylorme/eslint-annotate-action@v2
        with:
          report-json: eslint-report.json
```

---

## Prettier

Prettier — code formatter (công cụ định dạng code) — không kiểm tra logic, chỉ kiểm tra format. Trong CI chỉ **check** (không sửa), để đảm bảo developer đã format trước khi push.

### Check Format

```yaml
      - name: Check Prettier formatting
        run: npx prettier --check "src/**/*.{ts,tsx,js,json,css,scss}"
        # Fail nếu bất kỳ file nào chưa được format
```

### Kết Hợp ESLint + Prettier

```yaml
jobs:
  code-quality:
    name: Code Quality (Lint + Format)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Check Prettier
        run: npx prettier --check .

      - name: ESLint
        run: npx eslint . --max-warnings 0

      - name: TypeScript check
        run: npx tsc --noEmit        # Kiểm tra type mà không build output
```

---

## Python Linters

### ruff — Linter Nhanh Nhất Cho Python

ruff — Rust-based Python linter — thay thế flake8, isort, pyflakes trong một công cụ, nhanh hơn 100× so với flake8.

```yaml
      - name: Lint with ruff
        run: ruff check . --output-format=github
        # --output-format=github: tự động tạo GitHub annotations

      - name: Check format with ruff
        run: ruff format --check .
```

### black + isort + flake8 (Classic Stack)

```yaml
      - name: Check black formatting
        run: black --check --diff .

      - name: Check import order with isort
        run: isort --check-only --diff .

      - name: flake8 lint
        run: flake8 . --max-line-length=100 --exclude=migrations

      - name: mypy type checking
        run: mypy src --ignore-missing-imports
```

### pylint

```yaml
      - name: Run pylint
        run: pylint src --fail-under=8.0
        # Fail nếu score dưới 8/10
```

---

## Go / Java / Other Languages

### Go

```yaml
      - name: Go lint with golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m --out-format=github-actions
```

### Java / Checkstyle

```yaml
      - name: Run Checkstyle
        run: mvn checkstyle:check --no-transfer-progress

      - name: Run SpotBugs
        run: mvn spotbugs:check --no-transfer-progress
```

### Shell Scripts

```yaml
      - name: Lint shell scripts with ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          scandir: './scripts'
          severity: warning
```

### Dockerfile

```yaml
      - name: Lint Dockerfile with Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          failure-threshold: warning
```

### YAML / Markdown

```yaml
      - name: Lint YAML files
        uses: ibiqlik/action-yamllint@v3
        with:
          config_file: .yamllint.yml

      - name: Lint Markdown
        uses: DavidAnson/markdownlint-cli2-action@v16
        with:
          globs: '**/*.md'
```

---

## SonarQube / SonarCloud

SonarQube — SAST (Static Application Security Testing — Kiểm Tra Bảo Mật Ứng Dụng Tĩnh) và quality gate platform.
SonarCloud — phiên bản SaaS (Software as a Service — Phần Mềm Dạng Dịch Vụ) của SonarQube, miễn phí cho open source.

### SonarCloud

```yaml
jobs:
  sonar:
    name: SonarCloud Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # Full history để SonarCloud phân tích blame

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Cache SonarCloud packages
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Build and test
        run: mvn -B verify --no-transfer-progress

      - name: SonarCloud Scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}    # Cần để comment lên PR
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn -B sonar:sonar \
            -Dsonar.projectKey=my-org_my-project \
            -Dsonar.organization=my-org \
            -Dsonar.host.url=https://sonarcloud.io
```

### SonarQube Self-hosted

```yaml
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        with:
          args: >
            -Dsonar.projectKey=my-project
            -Dsonar.sources=src
            -Dsonar.tests=tests
            -Dsonar.python.coverage.reportPaths=coverage.xml

      - name: SonarQube Quality Gate
        uses: SonarSource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### SonarQube Quality Gate (Ngưỡng Chất Lượng)

Quality Gate — bộ điều kiện mà code mới phải đáp ứng để được merge. Nếu fail, CI fail.

```
Ví dụ Quality Gate "Sonar Way":
┌───────────────────────────────────────────┐
│  Conditions for "New Code" (Code mới)     │
│                                           │
│  ✅ Coverage on New Code ≥ 80%            │
│  ✅ Duplicated Lines on New Code < 3%     │
│  ✅ Maintainability Rating = A            │
│  ✅ Reliability Rating = A               │
│  ✅ Security Rating = A                  │
│  ✅ Security Hotspots Reviewed = 100%    │
└───────────────────────────────────────────┘
```

---

## Super-linter

Super-linter — một action chạy nhiều linters cho nhiều ngôn ngữ cùng lúc.

```yaml
jobs:
  super-lint:
    name: Super Lint
    runs-on: ubuntu-latest
    permissions:
      contents: read
      statuses: write         # Cần để update commit status
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Super-linter
        uses: super-linter/super-linter@v7
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BRANCH: main
          VALIDATE_ALL_CODEBASE: false    # false = chỉ lint file thay đổi (nhanh hơn)
          VALIDATE_JAVASCRIPT_ES: true
          VALIDATE_TYPESCRIPT_ES: true
          VALIDATE_PYTHON_RUFF: true
          VALIDATE_DOCKERFILE_HADOLINT: true
          VALIDATE_YAML: true
          VALIDATE_MARKDOWN: true
          LINTER_RULES_PATH: .github/linters    # Thư mục chứa config files
```

---

## Quality Gates và Thresholds

### PR Labels Dựa Trên Chất Lượng

```yaml
      - name: Add quality label
        uses: actions/github-script@v7
        if: failure()
        with:
          script: |
            github.rest.issues.addLabels({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: ['needs-fix']
            })
```

### Chặn Merge Khi Coverage Giảm

```yaml
      - name: Coverage diff check
        uses: actions/github-script@v7
        with:
          script: |
            const { execSync } = require('child_process');
            const currentCoverage = parseFloat(
              execSync('cat coverage-summary.json | jq .total.lines.pct').toString()
            );
            const baseCoverage = 80.0;    // Ngưỡng tối thiểu

            if (currentCoverage < baseCoverage) {
              core.setFailed(
                `Coverage ${currentCoverage}% is below minimum ${baseCoverage}%`
              );
            }
```

### Commit Message Linting

```yaml
      - name: Conventional Commits check
        uses: wagoid/commitlint-github-action@v6
        with:
          configFile: commitlint.config.js
          # Ví dụ config:
          # { extends: ['@commitlint/config-conventional'] }
          # Chấp nhận: feat:, fix:, docs:, chore:, refactor:, test:, ci:
```

---

## Auto-fix và Auto-commit

Tự động sửa lỗi format và commit lại — hữu ích nhưng cần cẩn thận để không tạo commit loop.

```yaml
jobs:
  auto-fix:
    runs-on: ubuntu-latest
    # Chỉ chạy trên PR (không chạy trên main để tránh direct commit)
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.PAT_TOKEN }}   # PAT để push commit
          ref: ${{ github.head_ref }}        # Checkout nhánh của PR

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Run Prettier auto-fix
        run: npx prettier --write .

      - name: Run ESLint auto-fix
        run: npx eslint . --fix

      - name: Commit fixes if any
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'style: auto-fix formatting'
          commit_author: 'github-actions[bot] <github-actions[bot]@users.noreply.github.com>'
```

### Danger Checks — Review Automation

Danger — công cụ tự động review PR dựa trên rules.

```yaml
      - name: Run Danger
        uses: danger/danger-action@v2
        with:
          danger_id: 'danger'
        env:
          DANGER_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Dangerfile.js định nghĩa rules:
          # - Warn nếu PR lớn hơn 500 dòng
          # - Fail nếu CHANGELOG không được cập nhật
          # - Warn nếu không có tests cho code mới
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác nhau giữa linter và formatter?

**Trả lời mẫu:**
- **Linter** (ESLint, flake8) — phân tích code để tìm lỗi logic, anti-patterns, unused variables, potential bugs. Quan tâm đến "code có đúng không".
- **Formatter** (Prettier, black) — chuẩn hóa style: indent, spacing, quotes, line length. Quan tâm đến "code có đẹp không". Formatter thường có thể auto-fix; linter thì không phải lúc nào cũng có thể.

### Câu 2: SonarQube Quality Gate là gì và tại sao quan trọng?

**Trả lời mẫu:**
Quality Gate là bộ điều kiện mà code mới phải đáp ứng — coverage tối thiểu, không có blocker/critical issues, security hotspots đã được review. Nếu Quality Gate fail, CI fail và merge bị chặn. Đây là cơ chế giữ kỷ luật kỹ thuật: không ai có thể merge code tệ dù quản lý có áp lực.

### Câu 3: Khi nào nên dùng Super-linter vs individual linters?

**Trả lời mẫu:**
- **Super-linter:** Phù hợp cho project đa ngôn ngữ, team muốn setup nhanh với ít config. Trade-off: chạy chậm hơn (nhiều linter), khó tune config cho từng ngôn ngữ.
- **Individual linters:** Kiểm soát chi tiết hơn, nhanh hơn vì chỉ chạy linter cần thiết, dễ update version độc lập. Phù hợp khi project chủ yếu một ngôn ngữ hoặc team có yêu cầu cụ thể.

---

## 📂 Điều Hướng

- [← Testing Strategies](2-testing-strategies.md)
- [→ Build & Artifacts](4-build-artifacts.md)
- [↑ Quay lại README](README.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
