# Testing Strategies — Chiến Lược Kiểm Thử Trong CI

> Test là trái tim của CI pipeline. GitHub Actions không chỉ chạy test — nó còn song song hóa, báo cáo kết quả, enforce coverage threshold, và ngăn code tệ merge vào nhánh chính.

## 📋 Mục Lục

1. [Pyramid Kiểm Thử](#pyramid-kiểm-thử)
2. [Unit Tests](#unit-tests)
3. [Integration Tests](#integration-tests)
4. [Code Coverage](#code-coverage)
5. [Test Parallelism](#test-parallelism)
6. [Test Reporting](#test-reporting)
7. [End-to-End Tests](#end-to-end-tests)
8. [Fail-fast vs Continue-on-error](#fail-fast-vs-continue-on-error)

---

## Pyramid Kiểm Thử

Test Pyramid (Kim Tự Tháp Kiểm Thử) — mô hình phân bổ loại test theo chi phí và tốc độ:

```
                    ▲
                   /E2E\              E2E Tests — ít nhất, chậm nhất, đắt nhất
                  /─────\            Playwright, Selenium, Cypress
                 / Integ  \
                /──────────\         Integration Tests — kiểm tra nhiều component
               /  Unit Tests \       cùng nhau, database, API calls thật
              /────────────────\
             /  Unit Tests       \   Unit Tests — nhiều nhất, nhanh nhất, rẻ nhất
            /────────────────────\   Isolated, mock dependencies

Nguyên tắc: 70% Unit | 20% Integration | 10% E2E
```

---

## Unit Tests

Unit test — kiểm tra từng function / class / module một cách độc lập, mock toàn bộ dependencies bên ngoài.

### Node.js / Jest

```yaml
jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Run unit tests
        run: npx jest --testPathPattern="unit" --coverage --coverageReporters=text,lcov

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: jest-coverage
          path: coverage/lcov.info
          retention-days: 7
```

### Python / pytest

```yaml
      - name: Run unit tests
        run: |
          pytest tests/unit \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v
```

### Go

```yaml
      - name: Run unit tests
        run: go test ./... -v -race -coverprofile=coverage.out

      - name: Generate coverage report
        run: go tool cover -html=coverage.out -o coverage.html

      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below threshold 80%"
            exit 1
          fi
```

### Java / JUnit

```yaml
      - name: Run unit tests
        run: mvn test -pl :unit-tests --no-transfer-progress

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: target/surefire-reports/**/*.xml
```

---

## Integration Tests

Integration test — kiểm tra nhiều component phối hợp: API + database, service + queue, v.v. Cần khởi động dependencies (database, cache) trong CI.

### Dùng Service Containers — Docker

GitHub Actions hỗ trợ **service containers** (container dịch vụ) — chạy Docker containers trong cùng network với runner job.

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest

    services:
      postgres:                         # Service tên "postgres"
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:                            # Service tên "redis"
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration
```

### MySQL + MongoDB

```yaml
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: >-
          --health-cmd "mysqladmin ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 10

      mongodb:
        image: mongo:7
        ports:
          - 27017:27017
```

### Chờ Service Sẵn Sàng (Wait for Readiness)

```yaml
      - name: Wait for services
        run: |
          until pg_isready -h localhost -p 5432; do
            echo "Waiting for PostgreSQL..."
            sleep 2
          done
          echo "PostgreSQL is ready"
```

### Docker Compose Cho Test Phức Tạp

```yaml
      - name: Start services with Docker Compose
        run: docker compose -f docker-compose.test.yml up -d

      - name: Wait for services
        run: docker compose -f docker-compose.test.yml run wait-for-services

      - name: Run integration tests
        run: npm run test:integration

      - name: Stop services
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

---

## Code Coverage

Code Coverage (Độ Bao Phủ Mã) — tỉ lệ phần trăm code được test chạy qua. Không phải mục tiêu, mà là safety net (lưới an toàn).

### Codecov Integration

```yaml
      - name: Run tests with coverage
        run: pytest --cov=src --cov-report=xml

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: ./coverage.xml
          flags: unittests            # Gắn nhãn để phân biệt loại test
          fail_ci_if_error: true      # Fail CI nếu upload lỗi
```

### Coveralls

```yaml
      - name: Coveralls
        uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          path-to-lcov: coverage/lcov.info
```

### Enforce Coverage Threshold Thủ Công

```yaml
      - name: Check coverage threshold
        run: |
          LINES=$(python -c "
          import xml.etree.ElementTree as ET
          tree = ET.parse('coverage.xml')
          root = tree.getroot()
          print(root.attrib['line-rate'])
          ")
          PERCENT=$(python -c "print(round(float('$LINES') * 100, 2))")
          echo "Line coverage: ${PERCENT}%"
          python -c "
          import sys
          coverage = float('$PERCENT')
          threshold = 80.0
          if coverage < threshold:
              print(f'Coverage {coverage}% is below threshold {threshold}%')
              sys.exit(1)
          print(f'Coverage {coverage}% meets threshold {threshold}%')
          "
```

### Jest Coverage Enforcement

```yaml
# jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: -10    // Cho phép tối đa 10 statements chưa được cover
    }
  }
}
```

---

## Test Parallelism

Parallelism (Song Song Hóa) — chạy test đồng thời để giảm tổng thời gian CI.

### Matrix Strategy — Chạy Song Song Theo Cấu Hình

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ['18', '20', '22']
        # Tạo ra 3 × 3 = 9 jobs chạy song song

      fail-fast: false          # Không dừng các jobs khác khi một job fail
      max-parallel: 6           # Giới hạn 6 jobs chạy cùng lúc (tiết kiệm cost)

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

### Sharding — Phân Mảnh Test

Sharding — chia bộ test thành N mảnh, chạy song song, giảm thời gian tuyến tính.

```yaml
# Jest sharding (Jest ≥ 28)
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]    # 4 shards
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Run shard ${{ matrix.shard }}/4
        run: npx jest --shard=${{ matrix.shard }}/4
```

```yaml
# Playwright sharding
      - name: Run Playwright shard ${{ matrix.shard }}/4
        run: npx playwright test --shard=${{ matrix.shard }}/4

      - name: Upload blob report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report
          retention-days: 1
```

```yaml
# Merge Playwright reports sau khi tất cả shards hoàn thành
  merge-reports:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Download all blob reports
        uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true

      - name: Merge reports
        run: npx playwright merge-reports --reporter html ./all-blob-reports

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        with:
          name: html-report
          path: playwright-report/
```

---

## Test Reporting

### JUnit XML Report

Hầu hết test frameworks có thể xuất JUnit XML format — GitHub Actions có thể đọc và hiển thị kết quả trên UI.

```yaml
      # Jest
      - run: npx jest --reporters=default --reporters=jest-junit
        env:
          JEST_JUNIT_OUTPUT_DIR: ./test-results

      # pytest
      - run: pytest --junitxml=test-results.xml

      # Go
      - run: go test ./... -v 2>&1 | go-junit-report > test-results.xml

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: test-results/**/*.xml
          check_name: 'Test Results'
          comment_mode: always       # Đăng comment lên PR
```

### Annotate Failed Tests Trên PR

```yaml
      - name: Test with annotations
        uses: actions/github-script@v7
        if: failure()
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('test-results.json'));
            for (const failure of results.failures) {
              core.error(failure.message, {
                file: failure.file,
                startLine: failure.line
              });
            }
```

---

## End-to-End Tests

E2E Tests — End-to-End Tests (Kiểm Thử Đầu Cuối) — kiểm tra toàn bộ ứng dụng qua UI hoặc API thật. Chậm nhưng có giá trị cao cho regression testing.

### Playwright

```yaml
  e2e:
    runs-on: ubuntu-latest
    needs: build          # Chỉ chạy E2E sau khi build xong
    timeout-minutes: 60

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000

      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### Cypress

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cypress run
        uses: cypress-io/github-action@v6
        with:
          build: npm run build
          start: npm start
          wait-on: 'http://localhost:3000'
          wait-on-timeout: 60
          browser: chrome
          record: true              # Record vào Cypress Cloud
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
```

---

## Fail-fast vs Continue-on-error

### fail-fast Trong Matrix

```yaml
strategy:
  fail-fast: true    # Mặc định — dừng tất cả matrix jobs khi một job fail
                     # Tiết kiệm cost nhưng mất kết quả từ jobs khác

  fail-fast: false   # Tiếp tục chạy tất cả matrix jobs dù có job fail
                     # Hữu ích khi cần biết test fail ở OS/version nào
```

### continue-on-error Cho Steps Không Bắt Buộc

```yaml
steps:
  - name: Run flaky E2E test
    run: npm run test:e2e
    continue-on-error: true     # Step fail nhưng job vẫn tiếp tục → trạng thái "neutral"

  - name: Run security scan
    run: trivy image my-app:latest
    continue-on-error: true     # Scan fail không block CI (có thể thay đổi sau khi fix)
```

### Retry Pattern Cho Flaky Tests

Flaky tests — test không ổn định — đôi khi pass đôi khi fail không vì lý do code mà vì timing, network, v.v.

```yaml
      - name: Run flaky tests with retry
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 10
          max_attempts: 3
          command: npm run test:e2e

      # Hoặc dùng retry trong Jest
      - run: npx jest --testNamePattern="flaky" --retryTimes=3
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Tại sao cần chạy test song song trong CI?

**Trả lời mẫu:**
Khi team lớn, có hàng trăm PR mỗi ngày. Nếu test mất 20 phút chạy tuần tự, developer phải chờ lâu mới biết code có pass không — làm chậm velocity (tốc độ phát triển). Sharding và matrix strategy có thể giảm từ 20 phút xuống còn 5 phút bằng cách chạy song song trên nhiều runners.

### Câu 2: Service containers hoạt động như thế nào?

**Trả lời mẫu:**
GitHub Actions khởi động Docker containers được khai báo trong `services:` và kết nối chúng vào cùng network với runner. Job có thể truy cập service qua `localhost:<port>`. GitHub tự động chờ health check pass trước khi bắt đầu steps. Sau khi job hoàn thành, containers được dọn dẹp tự động.

### Câu 3: Code coverage 100% có nghĩa là không có bug?

**Trả lời mẫu:**
Không. Coverage 100% chỉ có nghĩa là mọi dòng code đã được chạy qua trong test — không nghĩa là mọi edge case được kiểm tra. Một test có thể chạy qua code mà không assert gì và vẫn đạt 100% coverage. Ngưỡng thực tế tốt là 70–80% line coverage kết hợp với review chất lượng test.

### Câu 4: Phân biệt unit test và integration test trong CI?

**Trả lời mẫu:**
- **Unit test** — chạy nhanh (giây), không cần infrastructure, mock tất cả I/O. Chạy sớm nhất trong pipeline, fail nhanh.
- **Integration test** — cần database, cache, message queue thật (qua service containers). Chậm hơn (phút), nhưng kiểm tra rằng các components thực sự hoạt động cùng nhau.

**Tổ chức pipeline:** Unit test chạy trước, integration test chạy sau (`needs: unit-test`). Fail unit test dừng pipeline ngay, không lãng phí thời gian chạy integration test cho code đã sai.

---

## 📂 Điều Hướng

- [← Checkout & Setup](1-checkout-setup.md)
- [→ Linting & Code Quality](3-linting-quality.md)
- [↑ Quay lại README](README.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
