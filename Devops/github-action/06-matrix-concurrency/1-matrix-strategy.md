# Matrix Strategy — Chiến Lược Ma Trận

> Matrix Strategy cho phép một job chạy song song trên nhiều cấu hình khác nhau — test đa phiên bản, đa hệ điều hành, đa ngôn ngữ trong cùng một workflow định nghĩa.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#1-khái-niệm-cơ-bản)
2. [Cú Pháp Matrix](#2-cú-pháp-matrix)
3. [include — Bổ Sung Cấu Hình](#3-include--bổ-sung-cấu-hình)
4. [exclude — Loại Trừ Cấu Hình](#4-exclude--loại-trừ-cấu-hình)
5. [fail-fast và max-parallel](#5-fail-fast-và-max-parallel)
6. [Matrix Nhiều Chiều](#6-matrix-nhiều-chiều)
7. [Dynamic Matrix — Ma Trận Động](#7-dynamic-matrix--ma-trận-động)
8. [Thực Hành: Dự Án Thực Tế](#8-thực-hành-dự-án-thực-tế)
9. [Anti-Patterns — Các Lỗi Thường Gặp](#9-anti-patterns--các-lỗi-thường-gặp)
10. [Tóm Tắt](#10-tóm-tắt)

---

## 1. Khái Niệm Cơ Bản

### Matrix là gì?

**Matrix Strategy** (Chiến Lược Ma Trận) là cơ chế GitHub Actions cho phép bạn định nghĩa một job một lần, nhưng GitHub sẽ tự động tạo ra nhiều phiên bản (instances) của job đó với các tham số khác nhau, chạy song song.

```
Matrix Definition:
  node: [18, 20, 22]
  os:   [ubuntu-latest, windows-latest]

→ Tạo ra 6 job instances chạy đồng thời:
  ┌──────────────────────┬──────────────────────┐
  │ node=18, ubuntu      │ node=18, windows      │
  ├──────────────────────┼──────────────────────┤
  │ node=20, ubuntu      │ node=20, windows      │
  ├──────────────────────┼──────────────────────┤
  │ node=22, ubuntu      │ node=22, windows      │
  └──────────────────────┴──────────────────────┘
```

### Lợi Ích

| Không Dùng Matrix | Dùng Matrix |
|---|---|
| Phải viết lặp lại 6 jobs | Một job definition, tự nhân bản |
| Thay đổi phải sửa ở nhiều chỗ | Thay đổi một lần, áp dụng tất cả |
| Dễ quên cập nhật một chỗ nào đó | Nhất quán hoàn toàn |
| Tổng thời gian = tổng các jobs | Tổng thời gian ≈ job chậm nhất |

---

## 2. Cú Pháp Matrix

### Cú Pháp Cơ Bản

```yaml
name: Test Matrix

on: [push, pull_request]

jobs:
  test:
    name: Test Node ${{ matrix.node }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - run: npm ci
      - run: npm test
```

**Giải thích:**
- `strategy.matrix` — khai báo các chiều (dimensions) của ma trận
- `${{ matrix.node }}` — truy cập giá trị trong matrix context (ngữ cảnh ma trận)
- `matrix.os` — có thể dùng trong `runs-on` để chọn runner

### Tham Chiếu matrix Context

```yaml
steps:
  - name: Print matrix values
    run: |
      echo "Node version: ${{ matrix.node }}"
      echo "OS: ${{ matrix.os }}"
      echo "Python: ${{ matrix.python-version }}"

  # Dùng trong điều kiện
  - name: Windows-only step
    if: matrix.os == 'windows-latest'
    run: echo "Chỉ chạy trên Windows"

  # Dùng trong tên artifact
  - name: Upload artifact
    uses: actions/upload-artifact@v4
    with:
      name: build-${{ matrix.os }}-node${{ matrix.node }}
      path: dist/
```

---

## 3. include — Bổ Sung Cấu Hình

### include là gì?

`include` cho phép **thêm tham số bổ sung** vào một số combination cụ thể, hoặc **thêm combination mới** không có trong matrix gốc.

### Trường Hợp 1: Thêm Tham Số Vào Combination Có Sẵn

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    include:
      # Thêm biến 'experimental' chỉ cho combination os=ubuntu + node=20
      - os: ubuntu-latest
        node: 20
        experimental: true

steps:
  - name: Mark as experimental if applicable
    if: matrix.experimental == true
    run: echo "Đây là combination thử nghiệm"
```

### Trường Hợp 2: Thêm Combination Mới Hoàn Toàn

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    include:
      # Thêm combination mới: macos + node 22 (không có trong matrix gốc)
      - os: macos-latest
        node: 22
        # Có thể thêm bất kỳ biến nào
        experimental: true
```

**Kết quả:**
```
ubuntu + node18  ←  từ matrix gốc
ubuntu + node20  ←  từ matrix gốc (+ experimental=true từ include)
windows + node18 ←  từ matrix gốc
windows + node20 ←  từ matrix gốc
macos + node22   ←  thêm mới từ include
```

### Ví Dụ Thực Tế: Platform-Specific Configurations

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    include:
      - os: ubuntu-latest
        # Ubuntu dùng apt để cài dependencies
        install-cmd: sudo apt-get install -y libssl-dev
        artifact-ext: .tar.gz
      - os: windows-latest
        install-cmd: choco install openssl
        artifact-ext: .zip
      - os: macos-latest
        install-cmd: brew install openssl
        artifact-ext: .tar.gz

steps:
  - name: Install system dependencies
    run: ${{ matrix.install-cmd }}

  - name: Build
    run: make build

  - name: Upload
    uses: actions/upload-artifact@v4
    with:
      name: app-${{ matrix.os }}${{ matrix.artifact-ext }}
      path: dist/
```

---

## 4. exclude — Loại Trừ Cấu Hình

### exclude là gì?

`exclude` loại bỏ các combination cụ thể khỏi matrix — hữu ích khi một số combination không hợp lệ hoặc không cần thiết.

### Cú Pháp exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [16, 18, 20, 22]
    exclude:
      # Node 16 không hỗ trợ trên macOS runner mới nhất
      - os: macos-latest
        node: 16
      # Windows không cần test Node 16 (deprecated)
      - os: windows-latest
        node: 16
```

**Kết quả:** Từ 12 combinations ban đầu (3 × 4), sau khi exclude còn 10 combinations.

### include + exclude Kết Hợp

```yaml
strategy:
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, windows-latest]
    exclude:
      # Bỏ windows + node22 (chưa ổn định)
      - os: windows-latest
        node: 22
    include:
      # Thay thế bằng windows + node20 với flag đặc biệt
      - os: windows-latest
        node: 20
        windows-specific: true
```

---

## 5. fail-fast và max-parallel

### fail-fast — Dừng Nhanh Khi Có Lỗi

```yaml
strategy:
  fail-fast: true   # mặc định: true
  matrix:
    node: [18, 20, 22]
```

- **`fail-fast: true`** (mặc định): Nếu một job trong matrix thất bại, GitHub Actions hủy tất cả job đang chạy còn lại → **tiết kiệm minutes**
- **`fail-fast: false`**: Tất cả jobs chạy đến hết dù có lỗi → **xem đầy đủ kết quả**

**Khi nào dùng `fail-fast: false`?**

```yaml
# Khi muốn biết tất cả OS nào bị lỗi, không chỉ OS đầu tiên
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### max-parallel — Giới Hạn Song Song

```yaml
strategy:
  max-parallel: 3   # Chạy tối đa 3 jobs cùng lúc
  matrix:
    node: [14, 16, 18, 20, 22]
    os: [ubuntu-latest, windows-latest, macos-latest]
    # 15 combinations, nhưng chỉ 3 chạy cùng lúc
```

**Khi nào dùng `max-parallel`?**
- Self-hosted runners có giới hạn số lượng
- Tránh quá tải external service (database, API)
- Kiểm soát chi phí cloud resources

```yaml
# Ví dụ: Integration tests với shared database
strategy:
  max-parallel: 2   # Chỉ 2 jobs dùng DB cùng lúc tránh conflict
  fail-fast: false
  matrix:
    test-suite: [auth, users, orders, products, payments]
```

---

## 6. Matrix Nhiều Chiều

### Ma Trận 3D và Phức Tạp

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    database: [postgres, mysql, sqlite]
    # 2 × 2 × 3 = 12 combinations
```

### Ma Trận Với Objects (Đối Tượng Phức Tạp)

```yaml
strategy:
  matrix:
    config:
      - { name: "Node 18 + Postgres", node: 18, db: postgres, db-port: 5432 }
      - { name: "Node 20 + MySQL",    node: 20, db: mysql,    db-port: 3306 }
      - { name: "Node 22 + SQLite",   node: 22, db: sqlite,   db-port: 0    }

steps:
  - name: Run tests (${{ matrix.config.name }})
    run: npm test
    env:
      DB_TYPE: ${{ matrix.config.db }}
      DB_PORT: ${{ matrix.config.db-port }}
```

### Ví Dụ: Cross-language Testing

```yaml
name: Cross-language compatibility

strategy:
  matrix:
    runtime:
      - { lang: node,   version: "18", cmd: "node index.js"   }
      - { lang: node,   version: "20", cmd: "node index.js"   }
      - { lang: python, version: "3.11", cmd: "python main.py" }
      - { lang: python, version: "3.12", cmd: "python main.py" }
      - { lang: go,     version: "1.22", cmd: "go run main.go" }

steps:
  - uses: actions/checkout@v4

  - name: Setup ${{ matrix.runtime.lang }} ${{ matrix.runtime.version }}
    uses: actions/setup-${{ matrix.runtime.lang }}@v4
    with:
      ${{ matrix.runtime.lang }}-version: ${{ matrix.runtime.version }}

  - name: Run
    run: ${{ matrix.runtime.cmd }}
```

---

## 7. Dynamic Matrix — Ma Trận Động

### Vấn Đề

Đôi khi bạn không biết trước các giá trị trong matrix — chúng phụ thuộc vào kết quả của bước trước (ví dụ: danh sách microservices thay đổi).

### Giải Pháp: Job Sinh Ra Matrix

```yaml
jobs:
  # Job 1: Tìm danh sách services cần build
  discover:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.find.outputs.services }}
    steps:
      - uses: actions/checkout@v4
      - name: Find changed services
        id: find
        run: |
          # Tìm các service có thay đổi trong commit này
          SERVICES=$(git diff --name-only HEAD~1 HEAD \
            | grep '^services/' \
            | cut -d'/' -f2 \
            | sort -u \
            | jq -R -s -c 'split("\n")[:-1]')
          echo "services=$SERVICES" >> $GITHUB_OUTPUT

  # Job 2: Build song song các services tìm được
  build:
    needs: discover
    if: ${{ needs.discover.outputs.services != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        # Matrix lấy từ output của job trước
        service: ${{ fromJson(needs.discover.outputs.services) }}

    steps:
      - uses: actions/checkout@v4
      - name: Build ${{ matrix.service }}
        run: docker build -t myapp/${{ matrix.service }} services/${{ matrix.service }}/
```

### Dynamic Matrix Với fromJson()

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          # Tạo matrix JSON động từ script
          MATRIX=$(cat << 'EOF'
          {
            "include": [
              {"env": "staging",    "replicas": 2, "cpu": "0.5"},
              {"env": "production", "replicas": 5, "cpu": "1.0"}
            ]
          }
          EOF
          )
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  deploy:
    needs: setup
    strategy:
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ matrix.env }}
        run: |
          kubectl scale deployment myapp \
            --replicas=${{ matrix.replicas }} \
            -n ${{ matrix.env }}
```

---

## 8. Thực Hành: Dự Án Thực Tế

### Ví Dụ 1: Node.js Library Testing

```yaml
name: Node.js Library CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test / Node ${{ matrix.node }} / ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false   # Muốn xem kết quả tất cả combinations
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]
        include:
          # LTS mới nhất trên Ubuntu cần thêm coverage report
          - node: 22
            os: ubuntu-latest
            coverage: true
        exclude:
          # macOS runner đắt, chỉ test LTS cuối
          - os: macos-latest
            node: 18
          - os: macos-latest
            node: 20

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Upload coverage
        if: matrix.coverage == true
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  # Job tổng hợp — chỉ pass khi tất cả matrix jobs pass
  test-summary:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check test results
        if: needs.test.result != 'success'
        run: |
          echo "❌ Một hoặc nhiều matrix job thất bại"
          exit 1
      - run: echo "✅ Tất cả matrix jobs thành công"
```

### Ví Dụ 2: Multi-cloud Deployment Matrix

```yaml
name: Deploy to Multiple Clouds

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

jobs:
  deploy:
    strategy:
      fail-fast: false
      max-parallel: 2   # Deploy 2 cloud cùng lúc
      matrix:
        cloud:
          - provider: aws
            region: us-east-1
            role: arn:aws:iam::123456789:role/GitHubActionsRole
          - provider: gcp
            region: us-central1
            project: my-gcp-project
          - provider: azure
            region: eastus
            subscription: my-azure-sub

    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to ${{ matrix.cloud.provider }}
        run: |
          echo "Deploying to ${{ matrix.cloud.provider }} / ${{ matrix.cloud.region }}"
          ./scripts/deploy-${{ matrix.cloud.provider }}.sh
        env:
          CLOUD_REGION: ${{ matrix.cloud.region }}
```

---

## 9. Anti-Patterns — Các Lỗi Thường Gặp

### ❌ Matrix Quá Lớn Không Có max-parallel

```yaml
# Sai: 5 × 5 × 3 = 75 jobs chạy đồng thời → tốn kém, quá tải
strategy:
  matrix:
    node: [14, 16, 18, 20, 22]
    os: [ubuntu, windows, macos]
    database: [postgres, mysql, sqlite, mongodb, redis]
```

```yaml
# Đúng: Giới hạn và chỉ test combinations thực sự cần
strategy:
  max-parallel: 5
  matrix:
    node: [18, 22]           # LTS và latest
    os: [ubuntu-latest]      # Chỉ Linux cho unit tests
    include:
      - node: 20
        os: windows-latest   # Cross-platform check chỉ 1 version
```

### ❌ Dùng fail-fast: true Khi Muốn Xem Đầy Đủ Kết Quả

```yaml
# Sai: Nếu ubuntu fail, không biết windows hay macos có fail không
strategy:
  fail-fast: true  # mặc định
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

```yaml
# Đúng: Xem tất cả results để triage (phân loại) lỗi theo OS
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### ❌ Hard-code Values Có Thể Lỗi Thời

```yaml
# Sai: Node 16 EOL (End of Life — Hết Vòng Đời)
matrix:
  node: [14, 16, 18]
```

```yaml
# Đúng: Chỉ test supported versions, dùng comment để nhắc cập nhật
matrix:
  # LTS schedule: 18 (Active), 20 (Active), 22 (Current) — cập nhật mỗi 6 tháng
  node: [18, 20, 22]
```

### ❌ Không Đặt Tên Job Có Nghĩa

```yaml
# Sai: Tên job không thông tin
jobs:
  test:
    runs-on: ${{ matrix.os }}
```

```yaml
# Đúng: Tên job rõ ràng giúp đọc kết quả dễ dàng
jobs:
  test:
    name: "Test / ${{ matrix.node }} / ${{ matrix.os }}"
    runs-on: ${{ matrix.os }}
```

---

## 10. Tóm Tắt

### Cheat Sheet

```yaml
strategy:
  matrix:
    # Khai báo các chiều
    key1: [val1, val2, val3]
    key2: [valA, valB]

    # Thêm hoặc bổ sung combinations
    include:
      - key1: val1
        extra-var: extra-val   # thêm biến cho combination cụ thể
      - key1: newval           # thêm combination mới

    # Loại trừ combinations không cần thiết
    exclude:
      - key1: val1
        key2: valA

  fail-fast: false    # không dừng khi một job fail
  max-parallel: 4     # tối đa 4 jobs cùng lúc
```

### Khi Nào Dùng Matrix?

| Tình Huống | Nên Dùng Matrix? |
|---|---|
| Test trên nhiều Node.js versions | ✅ |
| Test trên nhiều OS | ✅ |
| Build cho nhiều platforms | ✅ |
| Deploy nhiều môi trường song song | ✅ với `max-parallel` |
| Chỉ có một cấu hình duy nhất | ❌ |
| Jobs có logic hoàn toàn khác nhau | ❌ |

### Câu Hỏi Phỏng Vấn

**Q: Matrix strategy khác gì với chạy nhiều jobs riêng biệt?**

> Matrix cho phép định nghĩa một lần, tự động tạo nhiều phiên bản — dễ maintain, nhất quán. Nếu thay đổi logic test, chỉ cần sửa một chỗ thay vì N jobs.

**Q: `include` vs `exclude` dùng khi nào?**

> `include` — thêm combination mới hoặc biến bổ sung cho combination đặc biệt. `exclude` — loại bỏ combination không hợp lệ hoặc không cần thiết để tránh lãng phí.

**Q: fail-fast: true hay false tốt hơn?**

> Phụ thuộc mục đích. `true` (mặc định): phát hiện lỗi nhanh, tiết kiệm minutes. `false`: xem đầy đủ kết quả để biết mức độ ảnh hưởng qua các OS/versions.

---

**Cập Nhật:** 2026-05-11 | **Tiếp Theo:** [2-concurrency.md](2-concurrency.md)
