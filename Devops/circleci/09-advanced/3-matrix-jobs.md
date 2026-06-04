# Matrix Jobs — Công Việc Ma Trận

> Matrix Jobs — Công Việc Ma Trận là tính năng của CircleCI cho phép chạy cùng một job với nhiều tổ hợp tham số khác nhau một cách đồng thời, mà không cần viết code lặp lại. Đây là cách hiệu quả nhất để test trên nhiều phiên bản ngôn ngữ, hệ điều hành, hoặc môi trường song song.

---

## 📋 Mục Lục

1. [Matrix Jobs Là Gì?](#1-matrix-jobs-là-gì)
2. [Cú Pháp Cơ Bản](#2-cú-pháp-cơ-bản)
3. [Matrix Với Nhiều Chiều Tham Số](#3-matrix-với-nhiều-chiều-tham-số)
4. [Matrix Nâng Cao: exclude Và alias](#4-matrix-nâng-cao-exclude-và-alias)
5. [Ví Dụ Thực Tế](#5-ví-dụ-thực-tế)
6. [Matrix Jobs Với Docker Images](#6-matrix-jobs-với-docker-images)
7. [Tổng Hợp Kết Quả Matrix — Fan-In](#7-tổng-hợp-kết-quả-matrix--fan-in)
8. [So Sánh Với Parallelism](#8-so-sánh-với-parallelism)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Matrix Jobs Là Gì?

Matrix Jobs là cách khai báo một job một lần nhưng CircleCI tự động tạo ra nhiều phiên bản của job đó với các tổ hợp tham số khác nhau.

### Ví Dụ Đơn Giản Nhất

**Không có Matrix (viết lặp):**

```yaml
# Cách cũ — phải viết lặp cho từng version
jobs:
  test:
    parameters:
      node-version:
        type: string

workflows:
  build:
    jobs:
      - test:
          name: test-node-18
          node-version: "18.0"
      - test:
          name: test-node-20
          node-version: "20.0"
      - test:
          name: test-node-22
          node-version: "22.0"
      # Thêm version mới → phải thêm thêm dòng → dễ quên, dễ sai
```

**Với Matrix (khai báo một lần):**

```yaml
workflows:
  build:
    jobs:
      - test:
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]
          # CircleCI tự tạo 3 job:
          #   test-18.0, test-20.0, test-22.0
```

---

## 2. Cú Pháp Cơ Bản

### Job Cần Có Parameters

```yaml
jobs:
  test:
    # Khai báo parameter mà matrix sẽ điều khiển
    parameters:
      node-version:
        type: string
      environment:
        type: string
        default: "test"

    docker:
      # Dùng parameter trong image name
      - image: cimg/node:<< parameters.node-version >>

    steps:
      - checkout
      - run:
          name: Hiển thị version
          command: node --version
      - run:
          name: Chạy kiểm thử
          command: npm test

workflows:
  multi-version-test:
    jobs:
      - test:
          matrix:
            parameters:
              # CircleCI tạo job cho mỗi giá trị trong list
              node-version: ["18.0", "20.0", "22.0"]
```

### Kết Quả Sinh Ra

CircleCI tự động tạo các job sau với tên được đặt tự động:
- `test-18.0`
- `test-20.0`
- `test-22.0`

---

## 3. Matrix Với Nhiều Chiều Tham Số

Khi khai báo nhiều parameter, CircleCI tạo ra **tích Cartesian — tích Đề-Các** (mọi tổ hợp có thể):

```yaml
jobs:
  integration-test:
    parameters:
      python-version:
        type: string
      database:
        type: string

    docker:
      - image: cimg/python:<< parameters.python-version >>
      - image: << parameters.database >>  # Database container

    steps:
      - checkout
      - run:
          name: Test với Python << parameters.python-version >> và << parameters.database >>
          command: pytest tests/integration/

workflows:
  cross-matrix-test:
    jobs:
      - integration-test:
          matrix:
            parameters:
              python-version: ["3.10", "3.11", "3.12"]
              database: ["postgres:14", "postgres:15", "mysql:8.0"]
```

**Kết quả:** 3 × 3 = **9 job** chạy song song:

```
integration-test-3.10-postgres:14
integration-test-3.10-postgres:15
integration-test-3.10-mysql:8.0
integration-test-3.11-postgres:14
integration-test-3.11-postgres:15
integration-test-3.11-mysql:8.0
integration-test-3.12-postgres:14
integration-test-3.12-postgres:15
integration-test-3.12-mysql:8.0
```

---

## 4. Matrix Nâng Cao: exclude Và alias

### exclude — Loại Trừ Tổ Hợp Cụ Thể

Không phải mọi tổ hợp đều có nghĩa. Dùng `exclude` để loại bỏ:

```yaml
workflows:
  test:
    jobs:
      - test:
          matrix:
            parameters:
              os: ["linux", "windows", "macos"]
              node-version: ["18.0", "20.0", "22.0"]
            exclude:
              # Node 18 không hỗ trợ trên macOS arm64 trong CI
              - os: "macos"
                node-version: "18.0"
              # Windows build chỉ cần test LTS version
              - os: "windows"
                node-version: "22.0"
```

**Kết quả:** Thay vì 9 job, chỉ tạo **7 job** (loại bỏ 2 tổ hợp không hợp lệ).

### alias — Đặt Tên Tùy Chỉnh Cho Matrix Job

Mặc định tên job tự động sinh ra có thể dài và khó đọc. Dùng `alias` để đặt tên riêng:

```yaml
workflows:
  build:
    jobs:
      - test:
          name: test-node-<< matrix.node-version >>  # ← Tên tùy chỉnh
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]
```

**Kết quả:** Job có tên `test-node-18.0`, `test-node-20.0`, `test-node-22.0` thay vì tên mặc định.

---

## 5. Ví Dụ Thực Tế

### Ví Dụ 1: Test Python Trên Nhiều Version

```yaml
version: 2.1

jobs:
  test-python:
    parameters:
      python-version:
        type: string

    docker:
      - image: cimg/python:<< parameters.python-version >>

    steps:
      - checkout
      - run:
          name: Cài đặt dependencies
          command: pip install -r requirements.txt
      - run:
          name: Kiểm thử unit
          command: pytest tests/unit/ -v --tb=short
      - run:
          name: Kiểm thử tích hợp
          command: pytest tests/integration/ -v
      - store_test_results:
          path: test-results

workflows:
  compatibility-matrix:
    jobs:
      - test-python:
          name: test-py-<< matrix.python-version >>
          matrix:
            parameters:
              python-version: ["3.9", "3.10", "3.11", "3.12"]
```

### Ví Dụ 2: Test Trên Nhiều OS

```yaml
version: 2.1

jobs:
  test-cross-platform:
    parameters:
      os:
        type: executor  # parameter kiểu executor
      node-version:
        type: string

    executor: << parameters.os >>

    steps:
      - checkout
      - run:
          name: Cài Node << parameters.node-version >>
          command: |
            # Cài NVM — Node Version Manager
            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
            export NVM_DIR="$HOME/.nvm"
            source "$NVM_DIR/nvm.sh"
            nvm install << parameters.node-version >>
      - run:
          name: Chạy kiểm thử
          command: npm test

executors:
  linux:
    machine:
      image: ubuntu-2204:current
  macos:
    macos:
      xcode: "15.0.0"

workflows:
  cross-platform:
    jobs:
      - test-cross-platform:
          matrix:
            parameters:
              os: ["linux", "macos"]
              node-version: ["18.20.0", "20.12.0"]
            exclude:
              - os: "macos"
                node-version: "18.20.0"  # Bỏ qua old LTS trên macOS
```

### Ví Dụ 3: Build Docker Image Cho Nhiều Architecture

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.6.0

jobs:
  build-docker:
    parameters:
      platform:
        type: string
      tag-suffix:
        type: string

    machine:
      image: ubuntu-2204:current

    steps:
      - checkout
      - run:
          name: Kích hoạt QEMU — Quick Emulator cho cross-platform build
          command: |
            docker run --privileged --rm tonistiigi/binfmt --install all
      - run:
          name: Build image cho << parameters.platform >>
          command: |
            docker buildx build \
              --platform << parameters.platform >> \
              --tag myapp:latest-<< parameters.tag-suffix >> \
              --push \
              .

workflows:
  multi-arch-build:
    jobs:
      - build-docker:
          matrix:
            parameters:
              platform: ["linux/amd64", "linux/arm64", "linux/arm/v7"]
              tag-suffix: ["amd64", "arm64", "armv7"]
            # Chỉ lấy tổ hợp đúng (diagonal — đường chéo)
            exclude:
              - platform: "linux/amd64"
                tag-suffix: "arm64"
              - platform: "linux/amd64"
                tag-suffix: "armv7"
              - platform: "linux/arm64"
                tag-suffix: "amd64"
              - platform: "linux/arm64"
                tag-suffix: "armv7"
              - platform: "linux/arm/v7"
                tag-suffix: "amd64"
              - platform: "linux/arm/v7"
                tag-suffix: "arm64"
```

---

## 6. Matrix Jobs Với Docker Images

Dùng parameter để thay đổi Docker image trong matrix:

```yaml
jobs:
  database-test:
    parameters:
      db-image:
        type: string
      db-port:
        type: integer

    docker:
      - image: cimg/python:3.12
      - image: << parameters.db-image >>
        environment:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_DATABASE: testdb

    environment:
      DB_PORT: << parameters.db-port >>

    steps:
      - checkout
      - run:
          name: Chờ database khởi động
          command: dockerize -wait tcp://localhost:<< parameters.db-port >> -timeout 60s
      - run:
          name: Chạy database migration
          command: python manage.py migrate
      - run:
          name: Kiểm thử với database << parameters.db-image >>
          command: pytest tests/db/

workflows:
  db-compatibility:
    jobs:
      - database-test:
          matrix:
            parameters:
              db-image: ["postgres:13", "postgres:14", "postgres:15"]
              db-port: [5432, 5432, 5432]
            exclude:
              # Giữ đúng pair image-port
              - db-image: "postgres:13"
                db-port: 5432
              # (giả lập: thực tế tất cả postgres dùng port 5432)
```

**Cách đơn giản hơn** với 1 parameter chứa thông tin đầy đủ:

```yaml
jobs:
  db-test:
    parameters:
      db-config:
        type: string  # "postgres:14:5432" hoặc "mysql:8.0:3306"

    steps:
      - run:
          name: Parse DB config
          command: |
            DB_IMAGE=$(echo "<< parameters.db-config >>" | cut -d: -f1-2)
            DB_PORT=$(echo "<< parameters.db-config >>" | cut -d: -f3)
            echo "export DB_IMAGE=$DB_IMAGE" >> $BASH_ENV
            echo "export DB_PORT=$DB_PORT" >> $BASH_ENV

workflows:
  db-matrix:
    jobs:
      - db-test:
          matrix:
            parameters:
              db-config:
                - "postgres:14:5432"
                - "postgres:15:5432"
                - "mysql:8.0:3306"
```

---

## 7. Tổng Hợp Kết Quả Matrix — Fan-In

Sau khi tất cả matrix job hoàn thành, thường cần một job tổng hợp (fan-in — thu gom):

```yaml
jobs:
  test:
    parameters:
      node-version:
        type: string
    docker:
      - image: cimg/node:<< parameters.node-version >>
    steps:
      - checkout
      - run: npm test

  # Job tổng hợp, chỉ chạy sau KHI TẤT CẢ matrix job xanh
  publish:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run:
          name: Publish lên npm
          command: npm publish

workflows:
  test-and-publish:
    jobs:
      - test:
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]

      - publish:
          requires:
            # Syntax đặc biệt để require toàn bộ matrix jobs
            - test-18.0
            - test-20.0
            - test-22.0
          filters:
            branches:
              only: main
```

**Cách linh hoạt hơn** dùng `matrix` trong `requires`:

```yaml
# Khi số lượng matrix value thay đổi thường xuyên, hardcode tên job dễ bị sai
# Giải pháp: dùng tên job với alias để dễ reference

workflows:
  ci:
    jobs:
      - test:
          name: test-node-<< matrix.node-version >>
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]

      - all-tests-passed:
          requires:
            - test-node-18.0
            - test-node-20.0
            - test-node-22.0
```

---

## 8. So Sánh Với Parallelism

Matrix Jobs và Parallelism — Song Song Hóa giải quyết các bài toán khác nhau:

| Tiêu Chí | Matrix Jobs | Parallelism |
| -------- | ----------- | ----------- |
| **Mục đích** | Chạy cùng job với tham số khác nhau | Chia nhỏ một job thành nhiều phần |
| **Số lượng job** | N job riêng biệt (N = số tổ hợp) | 1 job được nhân bản N lần |
| **Test isolation** | Mỗi job hoàn toàn độc lập | Các shard — mảnh chia sẻ test suite |
| **Dùng khi** | Test nhiều version, OS, DB | Test suite lớn cần chia nhỏ |
| **Artifacts** | Mỗi job có artifacts riêng | Cần merge artifacts từ các shard |
| **Cấu hình** | `matrix: parameters:` trong workflow | `parallelism: N` trong job |

### Kết Hợp Cả Hai

```yaml
jobs:
  test:
    parameters:
      node-version:
        type: string
    parallelism: 4  # Mỗi matrix job còn được chia thành 4 shard
    docker:
      - image: cimg/node:<< parameters.node-version >>
    steps:
      - checkout
      - run:
          command: |
            circleci tests glob "src/**/*.test.js" | \
            circleci tests split --split-by=timings | \
            xargs npx jest

workflows:
  full-matrix:
    jobs:
      - test:
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]
          # Kết quả: 3 version × 4 shard = 12 container chạy song song
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Matrix Jobs trong CircleCI là gì và khi nào nên dùng?**

> Matrix Jobs cho phép chạy cùng một job definition với nhiều tổ hợp parameter khác nhau đồng thời. Nên dùng khi: (1) cần test trên nhiều version ngôn ngữ (Python 3.9/3.10/3.11), (2) cross-platform testing — kiểm thử đa nền tảng (Linux/macOS/Windows), (3) test tương thích với nhiều phiên bản database, (4) build Docker image cho nhiều CPU architecture.

**Q: Sự khác biệt giữa Matrix Jobs và Parallelism là gì?**

> Matrix Jobs tạo ra nhiều job độc lập với cấu hình khác nhau — mỗi job là một biến thể của template job. Parallelism nhân bản một job thành nhiều bản giống hệt nhau để chia nhỏ workload (thường là test suite). Matrix = different configurations; Parallelism = same configuration, split work.

**Q: Làm thế nào để một job phụ thuộc vào TẤT CẢ matrix job hoàn thành?**

> Dùng tên cụ thể của từng matrix job trong `requires`. Nếu dùng `name` alias thì reference bằng tên alias đó. Ví dụ: `requires: [test-node-18.0, test-node-20.0, test-node-22.0]`. Khi thêm version mới vào matrix, cũng phải cập nhật `requires` tương ứng — đây là nhược điểm của cách tiếp cận này.

**Q: Làm thế nào để loại trừ một tổ hợp không hợp lệ trong matrix?**

> Dùng `exclude` trong `matrix` definition. Ví dụ:
> ```yaml
> matrix:
>   parameters:
>     os: ["linux", "macos"]
>     version: ["old", "new"]
>   exclude:
>     - os: "macos"
>       version: "old"  # macOS không hỗ trợ old version
> ```

---

## 🔗 Xem Thêm

- [4-self-hosted-runner.md](4-self-hosted-runner.md) — Chạy matrix job trên self-hosted runner
- [05-optimization/4-parallelism.md](../05-optimization/4-parallelism.md) — So sánh và kết hợp parallelism
- [CircleCI Docs: Matrix Jobs](https://circleci.com/docs/configuration-reference/#matrix-requires-version-21)

---

**Cập Nhật Lần Cuối:** 2026-05-20
