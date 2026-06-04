# 🔎 Code Scanning — Quét Mã Bảo Mật Trong CI

> Code Scanning — Quét Mã Bảo Mật là tích hợp các công cụ phân tích mã nguồn tĩnh (SAST — Static Application Security Testing) và động (DAST — Dynamic Application Security Testing) trực tiếp vào CI pipeline để phát hiện lỗ hổng bảo mật trước khi code lên production.

---

## 📚 Mục Lục

1. [Code Scanning Là Gì?](#code-scanning-là-gì)
2. [CodeQL — Công Cụ Phân Tích Mã Chính Thức](#codeql)
3. [Cấu Hình CodeQL Nâng Cao](#cấu-hình-codeql-nâng-cao)
4. [Semgrep — SAST Linh Hoạt](#semgrep)
5. [Snyk — Quét Cả Code và Dependencies](#snyk)
6. [DAST Với OWASP ZAP](#dast-với-owasp-zap)
7. [Trivy — Quét Container Images](#trivy)
8. [Tổng Hợp Kết Quả SARIF](#tổng-hợp-kết-quả-sarif)
9. [Security Gate — Chặn Merge Khi Có Lỗ Hổng](#security-gate)

---

## Code Scanning Là Gì?

### Phân Loại Các Loại Scanning

```
Code Scanning
├── SAST (Static Application Security Testing — Kiểm Tra Bảo Mật Tĩnh)
│   ├── Phân tích source code không cần chạy
│   ├── Phát hiện: SQL injection, XSS, command injection, hardcoded secrets
│   └── Công cụ: CodeQL, Semgrep, SonarQube, Checkmarx
│
├── SCA (Software Composition Analysis — Phân Tích Thành Phần Phần Mềm)
│   ├── Kiểm tra dependencies có lỗ hổng đã biết
│   ├── Phát hiện: CVE, GHSA, outdated packages
│   └── Công cụ: Snyk, OWASP Dependency Check, Trivy
│
├── Container Scanning (Quét Container)
│   ├── Quét Docker images, base images
│   ├── Phát hiện: OS vulnerabilities, outdated packages trong image
│   └── Công cụ: Trivy, Snyk Container, Clair
│
└── DAST (Dynamic Application Security Testing — Kiểm Tra Bảo Mật Động)
    ├── Tấn công ứng dụng đang chạy để tìm lỗ hổng
    ├── Phát hiện: Runtime vulnerabilities, misconfigurations
    └── Công cụ: OWASP ZAP, Burp Suite Enterprise
```

### SARIF — Static Analysis Results Interchange Format

SARIF — Định dạng Trao đổi Kết quả Phân tích Tĩnh là định dạng JSON chuẩn để report kết quả security scanning. GitHub hiểu SARIF và hiển thị kết quả trong Security tab.

---

## CodeQL

### CodeQL Là Gì?

**CodeQL** là công cụ phân tích code của GitHub, sử dụng ngôn ngữ truy vấn đặc biệt để tìm lỗ hổng bảo mật. Được tích hợp sẵn và miễn phí cho public repos.

**Ngôn ngữ hỗ trợ:** C/C++, C#, Go, Java/Kotlin, JavaScript/TypeScript, Python, Ruby, Swift

### Cấu Hình CodeQL Cơ Bản

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    # Chạy hàng tuần vào 2:00 sáng thứ Hai (UTC)
    - cron: '0 2 * * 1'

permissions:
  actions: read
  contents: read
  security-events: write  # Để upload kết quả lên GitHub Security

jobs:
  analyze:
    name: Phân tích CodeQL
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        language: ['javascript', 'python']
        # Thêm ngôn ngữ nếu repo có nhiều ngôn ngữ

    steps:
      - name: Checkout repository
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Khởi tạo CodeQL
        uses: github/codeql-action/init@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        with:
          languages: ${{ matrix.language }}
          # Dùng query suite mở rộng (tìm nhiều hơn nhưng có thể nhiều false positives)
          queries: +security-extended

      # Bước autobuild tự động build code (cần cho C/C++, C#, Java)
      - name: Autobuild
        uses: github/codeql-action/autobuild@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15

      - name: Chạy phân tích CodeQL
        uses: github/codeql-action/analyze@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        with:
          category: "/language:${{ matrix.language }}"
          # Upload kết quả lên GitHub Security tab
          upload: true
```

---

## Cấu Hình CodeQL Nâng Cao

### Custom Query Suite

```yaml
# Tạo file .github/codeql/codeql-config.yml
name: "Custom CodeQL Config"

queries:
  - uses: security-and-quality   # Query suite có sẵn
  - uses: ./custom-queries        # Custom queries của team

query-filters:
  - exclude:
      id: js/unused-local-variable  # Loại trừ rule cụ thể

paths:
  - src/                           # Chỉ scan thư mục src
  - lib/

paths-ignore:
  - src/generated/                 # Bỏ qua code được generate tự động
  - node_modules/
  - vendor/
  - "**/*.test.js"
  - "**/*.spec.ts"
```

```yaml
# Trong workflow, tham chiếu config file
- name: Initialize CodeQL
  uses: github/codeql-action/init@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
  with:
    languages: javascript
    config-file: .github/codeql/codeql-config.yml
```

### CodeQL Cho Java/Maven Build

```yaml
- name: Initialize CodeQL
  uses: github/codeql-action/init@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
  with:
    languages: java

# Không dùng autobuild — tự build để đảm bảo đúng cấu hình
- name: Setup Java
  uses: actions/setup-java@c5195efecf7bdfc987ee8bae7a71cb8b11521c00 # v4.7.1
  with:
    distribution: 'temurin'
    java-version: '21'

- name: Build với Maven
  run: mvn clean compile -B

- name: Analyze
  uses: github/codeql-action/analyze@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
```

---

## Semgrep

### Semgrep Là Gì?

**Semgrep** là SAST tool mã nguồn mở với cú pháp viết rules đơn giản, hỗ trợ 30+ ngôn ngữ và có registry hàng nghìn rules sẵn có.

### Tích Hợp Semgrep Vào CI

```yaml
# .github/workflows/semgrep.yml
name: Semgrep SAST Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  semgrep:
    name: Semgrep Scan
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Chạy Semgrep
        uses: semgrep/semgrep-action@713efdd345f3035192eaa63f56867b88e63e4e5d # v1
        with:
          # Dùng các rule sets phổ biến
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/nodejs-security
          # Hoặc chỉ định file rules riêng:
          # config: .semgrep/rules.yml

          # Xuất kết quả SARIF để upload GitHub Security
          generateSarif: "1"

      - name: Upload SARIF lên GitHub Security
        uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        if: always()
        with:
          sarif_file: semgrep.sarif
```

### Viết Custom Semgrep Rules

```yaml
# .semgrep/custom-rules.yml
rules:
  - id: no-hardcoded-credentials
    patterns:
      - pattern: |
          $VAR = "..."
      - metavariable-regex:
          metavariable: $VAR
          regex: '(?i)(password|passwd|secret|api_key|token)'
    message: |
      Phát hiện hardcoded credential trong biến '$VAR'.
      Dùng environment variables hoặc secrets management.
    languages: [javascript, typescript, python]
    severity: ERROR
    metadata:
      category: security
      cwe: CWE-798

  - id: no-sql-string-concat
    pattern: |
      $DB.query("..." + $INPUT)
    message: SQL injection risk — dùng parameterized queries
    languages: [javascript, typescript]
    severity: ERROR
```

---

## Snyk

### Snyk — Quét Cả Code, Dependencies và Container

```yaml
# .github/workflows/snyk.yml
name: Snyk Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  snyk-code:
    name: Snyk Code (SAST)
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Chạy Snyk để quét code
        uses: snyk/actions/node@b98d498629f1c5e001b5bcc1c4be52f3ec4a7c5a # v0.4.0
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: code test
          args: --severity-threshold=high --sarif-file-output=snyk-code.sarif
        continue-on-error: true  # Không fail pipeline, chỉ report

      - name: Upload kết quả Snyk Code lên GitHub Security
        uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        with:
          sarif_file: snyk-code.sarif

  snyk-deps:
    name: Snyk Open Source (SCA)
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Quét dependencies với Snyk
        uses: snyk/actions/node@b98d498629f1c5e001b5bcc1c4be52f3ec4a7c5a # v0.4.0
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --all-projects

  snyk-container:
    name: Snyk Container
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Build Docker image để scan
        run: docker build -t my-app:test .

      - name: Quét Docker image với Snyk
        uses: snyk/actions/docker@b98d498629f1c5e001b5bcc1c4be52f3ec4a7c5a # v0.4.0
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: my-app:test
          args: --severity-threshold=high --file=Dockerfile
        continue-on-error: true
```

---

## DAST Với OWASP ZAP

### OWASP ZAP — Zed Attack Proxy — Proxy Tấn Công Zed

DAST testing với OWASP ZAP chạy các cuộc tấn công thực tế vào ứng dụng đang chạy để tìm lỗ hổng runtime.

```yaml
# .github/workflows/dast.yml
name: DAST Security Testing

on:
  push:
    branches: [main]
  # Chạy hàng tuần — DAST mất thời gian hơn SAST
  schedule:
    - cron: '0 3 * * 0'  # Chủ Nhật 3:00 sáng UTC

permissions:
  contents: read
  security-events: write
  issues: write  # Để tạo issue nếu phát hiện lỗ hổng

jobs:
  dast:
    name: DAST với OWASP ZAP
    runs-on: ubuntu-latest
    environment: staging  # Chỉ test trên staging, không bao giờ production

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Khởi động ứng dụng (staging)
        run: |
          docker-compose -f docker-compose.staging.yml up -d
          # Đợi service sẵn sàng
          timeout 60 bash -c 'until curl -sf http://localhost:3000/health; do sleep 2; done'

      - name: ZAP Baseline Scan (quét cơ bản — nhanh)
        uses: zaproxy/action-baseline@7d786b5e0a2c3bc4e30bcc386f82b6a1efb5b4e0 # v0.14.0
        with:
          target: 'http://localhost:3000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'  # Ajax Spider để quét SPA (Single Page Application)
          issue_title: 'ZAP Baseline Scan Report'
          fail_action: warn   # Cảnh báo nhưng không fail

      # Hoặc dùng Full Scan (chậm hơn, toàn diện hơn)
      # - name: ZAP Full Scan
      #   uses: zaproxy/action-full-scan@...
      #   with:
      #     target: 'http://localhost:3000'

      - name: Dừng containers sau khi test
        if: always()
        run: docker-compose -f docker-compose.staging.yml down
```

---

## Trivy — Quét Container Images

### Trivy — Công Cụ Quét Bảo Mật Container Đa Năng

```yaml
# .github/workflows/trivy.yml
name: Trivy Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  trivy-fs:
    name: Trivy Filesystem Scan
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Quét filesystem với Trivy
        uses: aquasecurity/trivy-action@6e7b7d1fd3e4fef0c5fa8cce1229c54b2c9bd0d8 # v0.30.0
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-fs-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'           # Fail pipeline nếu có CRITICAL/HIGH

      - name: Upload SARIF lên GitHub Security
        uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        if: always()
        with:
          sarif_file: 'trivy-fs-results.sarif'

  trivy-container:
    name: Trivy Container Scan
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .

      - name: Quét Docker image với Trivy
        uses: aquasecurity/trivy-action@6e7b7d1fd3e4fef0c5fa8cce1229c54b2c9bd0d8 # v0.30.0
        with:
          image-ref: 'my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-image-results.sarif'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true     # Bỏ qua lỗ hổng chưa có bản vá

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
        if: always()
        with:
          sarif_file: 'trivy-image-results.sarif'
```

---

## Tổng Hợp Kết Quả SARIF

### Upload Nhiều SARIF Files

```yaml
- name: Upload tất cả SARIF results
  uses: github/codeql-action/upload-sarif@45775bd8235c68ba998cffa5171334d58593da47 # v3.28.15
  if: always()
  with:
    # Có thể upload một thư mục chứa nhiều SARIF files
    sarif_file: sarif-results/
    category: "security-scans"
    wait-for-processing: true  # Đợi GitHub xử lý xong
```

### Tổng Hợp Nhiều Công Cụ Trong Một Workflow

```yaml
name: Full Security Scan Suite

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'

permissions:
  contents: read
  security-events: write
  actions: read

jobs:
  codeql:
    name: CodeQL
    uses: ./.github/workflows/_codeql.yml

  semgrep:
    name: Semgrep
    uses: ./.github/workflows/_semgrep.yml

  trivy:
    name: Trivy
    uses: ./.github/workflows/_trivy.yml

  # Tổng hợp kết quả và notify
  security-summary:
    name: Security Summary
    needs: [codeql, semgrep, trivy]
    runs-on: ubuntu-latest
    if: always()

    steps:
      - name: Kiểm tra kết quả tổng thể
        run: |
          if [ "${{ needs.codeql.result }}" == "failure" ] || \
             [ "${{ needs.semgrep.result }}" == "failure" ] || \
             [ "${{ needs.trivy.result }}" == "failure" ]; then
            echo "::error::Phát hiện lỗ hổng bảo mật — xem GitHub Security tab"
            exit 1
          fi
          echo "✅ Tất cả security scans passed"
```

---

## Security Gate — Chặn Merge Khi Có Lỗ Hổng

### Cấu Hình Branch Protection Rules

```
GitHub Settings → Branches → Branch protection rules
  → Require status checks to pass before merging:
      ✅ CodeQL Analysis (javascript)
      ✅ Semgrep Scan
      ✅ Trivy Container Scan
  → Require code owner reviews
```

### Fail Pipeline Dựa Trên Severity

```yaml
- name: Kiểm tra kết quả bảo mật
  run: |
    # Đọc SARIF và fail nếu có findings nghiêm trọng
    CRITICAL_COUNT=$(cat trivy-results.sarif \
      | jq '[.runs[].results[] | select(.level == "error")] | length')

    if [ "$CRITICAL_COUNT" -gt "0" ]; then
      echo "::error::Phát hiện $CRITICAL_COUNT lỗ hổng CRITICAL/HIGH"
      echo "Xem chi tiết tại GitHub Security tab"
      exit 1
    fi

    echo "✅ Không phát hiện lỗ hổng nghiêm trọng"
```

---

## 📊 So Sánh Công Cụ Code Scanning

| Công Cụ | Loại | Ngôn Ngữ | Miễn Phí | GitHub Integration |
|---|---|---|---|---|
| **CodeQL** | SAST | 10+ | ✅ Public repos | ✅ Native |
| **Semgrep** | SAST | 30+ | ✅ Community | ✅ SARIF |
| **Snyk** | SAST + SCA | 20+ | ✅ Giới hạn | ✅ SARIF |
| **Trivy** | SCA + Container | N/A | ✅ Hoàn toàn | ✅ SARIF |
| **OWASP ZAP** | DAST | N/A | ✅ Hoàn toàn | ✅ Qua SARIF |
| **SonarQube** | SAST + Quality | 30+ | ✅ Community | ✅ SARIF |

---

## 🔗 Xem Thêm

- [5-secret-scanning.md](./5-secret-scanning.md) — Phát hiện secrets bị lộ
- [3-supply-chain.md](./3-supply-chain.md) — Bảo vệ dependencies
- [6-security-hardening.md](./6-security-hardening.md) — Checklist toàn diện

---

**Cập Nhật Lần Cuối:** 2026-05-12
