# 🔧 Jobs và Steps Trong CircleCI

> Job — Công Việc là đơn vị thực thi cơ bản; Steps — Các Bước là những hành động tuần tự bên trong job. Hiểu sâu về jobs và steps giúp xây dựng pipeline ổn định và hiệu quả.

---

## 📋 Mục Lục

1. [Job Là Gì?](#1-job-là-gì)
2. [Cấu Trúc Job](#2-cấu-trúc-job)
3. [Các Loại Executor Inline](#3-các-loại-executor-inline)
4. [Built-in Steps — Bước Tích Hợp Sẵn](#4-built-in-steps--bước-tích-hợp-sẵn)
5. [Step Điều Kiện — Conditional Steps](#5-step-điều-kiện--conditional-steps)
6. [Xử Lý Lỗi Trong Steps](#6-xử-lý-lỗi-trong-steps)
7. [Artifacts và Test Results](#7-artifacts-và-test-results)
8. [Resource Class — Lớp Tài Nguyên](#8-resource-class--lớp-tài-nguyên)
9. [Ví Dụ Thực Tế](#9-ví-dụ-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Job Là Gì?

**Job** là đơn vị thực thi độc lập trong CircleCI. Mỗi job:

- Chạy trong môi trường **hoàn toàn cách ly** (container Docker hoặc VM riêng)
- **Không chia sẻ filesystem** với jobs khác (phải dùng Workspace để truyền file)
- Có thể chạy **song song** với jobs khác trong cùng workflow
- Bao gồm một danh sách các **Steps** thực thi tuần tự từ trên xuống

```
Workflow
├── job-A  (chạy độc lập)
│   ├── step 1: checkout
│   ├── step 2: install deps
│   └── step 3: run tests
│
└── job-B  (chạy song song với job-A, hoặc sau khi job-A xong)
    ├── step 1: checkout
    └── step 2: run linting
```

---

## 2. Cấu Trúc Job

```yaml
jobs:
  my-job:
    # ── Executor (BẮT BUỘC) ──────────────────────────────
    docker:
      - image: cimg/node:20.0
        auth:                           # Xác thực Docker Hub (nếu cần)
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_TOKEN

    # ── Cấu hình tài nguyên ──────────────────────────────
    resource_class: medium              # Lớp tài nguyên CPU/RAM
    parallelism: 1                      # Số luồng song song (mặc định: 1)
    
    # ── Môi trường làm việc ──────────────────────────────
    working_directory: ~/project        # Thư mục làm việc
    shell: /bin/bash -eo pipefail       # Shell và options
    environment:                        # Biến môi trường cấp job
      NODE_ENV: test
      CI: "true"
    
    # ── Timeout ──────────────────────────────────────────
    # Không có timeout ở cấp job; dùng no_output_timeout trong run steps
    
    # ── Parameters ───────────────────────────────────────
    parameters:
      run-integration-tests:
        type: boolean
        default: false
    
    # ── Steps (BẮT BUỘC) ─────────────────────────────────
    steps:
      - checkout
      - run:
          name: Cài đặt dependencies
          command: npm ci
      - run:
          name: Chạy unit tests
          command: npm test
```

---

## 3. Các Loại Executor Inline

Executor định nghĩa **môi trường chạy job**. Có thể khai báo inline trong job hoặc tái sử dụng từ block `executors:`.

### 3.1 Docker Executor

Phổ biến nhất. Job chạy trong Docker container — thùng chứa ứng dụng.

```yaml
jobs:
  build:
    docker:
      # Image chính — container chạy steps
      - image: cimg/node:20.0
        auth:
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_TOKEN
        environment:
          NODE_ENV: test
      
      # Service containers — container dịch vụ phụ (accessible qua localhost)
      - image: cimg/postgres:14.0
        environment:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
      
      - image: redis:7-alpine
        # Redis accessible tại localhost:6379

    steps:
      - checkout
      - run:
          name: Chờ PostgreSQL sẵn sàng
          command: dockerize -wait tcp://localhost:5432 -timeout 1m
```

**Khi nào dùng Docker executor:**
- Ứng dụng web, API, microservices
- Cần service containers (DB, cache, message queue)
- Build nhanh với lightweight containers

### 3.2 Machine Executor — Máy Ảo Linux

Job chạy trên **Linux VM đầy đủ** — thường dùng khi cần Docker-in-Docker hoặc quyền truy cập hardware đặc biệt.

```yaml
jobs:
  docker-build:
    machine:
      image: ubuntu-2204:current        # Ubuntu 22.04 LTS
      docker_layer_caching: true        # DLC — Docker Layer Caching — Bộ Đệm Layer Docker
    
    resource_class: medium              # medium | large | xlarge | 2xlarge
    
    steps:
      - checkout
      - run:
          name: Build và push Docker image
          command: |
            docker build -t myapp:$CIRCLE_SHA1 .
            docker push myapp:$CIRCLE_SHA1
```

**Khi nào dùng Machine executor:**
- Build Docker images (cần Docker daemon)
- Cần privileged mode — chế độ đặc quyền
- Test với Docker Compose
- Cần `/dev/` access hoặc kernel features

### 3.3 macOS Executor

Chạy trên macOS — dành cho iOS/macOS development.

```yaml
jobs:
  ios-build:
    macos:
      xcode: "15.2.0"                   # Phiên bản Xcode
    
    resource_class: macos.m1.medium.gen1  # Apple Silicon
    
    steps:
      - checkout
      - run: pod install                 # CocoaPods — Quản Lý Phụ Thuộc iOS
      - run: xcodebuild test -scheme MyApp
```

### 3.4 Windows Executor

Chạy trên Windows Server — dành cho .NET và Windows-specific builds.

```yaml
jobs:
  windows-test:
    machine:
      image: windows-server-2022-gui:current
      shell: powershell.exe
    
    resource_class: windows.medium
    
    steps:
      - checkout
      - run: dotnet test
```

---

## 4. Built-in Steps — Bước Tích Hợp Sẵn

### 4.1 `checkout`

Lấy source code từ VCS — Version Control System (Git) về môi trường làm việc.

```yaml
steps:
  - checkout                   # Checkout vào working_directory

  # Tùy chỉnh path (ít dùng)
  - checkout:
      path: /tmp/repo          # Checkout vào path khác
```

> `checkout` tự động thiết lập Git credentials và clone repo. Nó **không** fetch toàn bộ lịch sử Git — chỉ shallow clone để nhanh hơn.

### 4.2 `run` — Chạy Lệnh Shell

Step quan trọng nhất. Thực thi lệnh trong shell.

```yaml
steps:
  # Dạng ngắn gọn
  - run: npm install

  # Dạng đầy đủ
  - run:
      name: Tên bước hiển thị trên UI
      command: |
        echo "Biến môi trường: $MY_VAR"
        npm run build
      environment:             # Biến môi trường chỉ trong run này
        MY_VAR: hello
        NODE_OPTIONS: "--max-old-space-size=4096"
      when: always             # Chạy ngay cả khi step trước thất bại
      no_output_timeout: 15m   # Timeout nếu không có output sau 15 phút
      working_directory: ./subdir  # Thư mục làm việc riêng cho step này
      shell: /bin/sh           # Override shell
      background: false        # Nếu true, step chạy nền (không chờ kết thúc)
```

**Tùy chọn `when`:**
- `on_success` (mặc định): Chỉ chạy nếu các steps trước thành công
- `on_fail`: Chỉ chạy nếu có step trước thất bại (dùng cho cleanup, notification)
- `always`: Luôn chạy bất kể kết quả

### 4.3 `save_cache` và `restore_cache`

Lưu và phục hồi cache — bộ nhớ đệm giữa các lần chạy pipeline.

```yaml
steps:
  # Phục hồi cache — luôn đặt TRƯỚC khi install
  - restore_cache:
      keys:
        # Khóa chính xác nhất (checksum của file lock)
        - node-v1-{{ checksum "package-lock.json" }}
        # Fallback — khóa dự phòng (prefix match)
        - node-v1-

  - run: npm ci

  # Lưu cache — sau khi install xong
  - save_cache:
      key: node-v1-{{ checksum "package-lock.json" }}
      paths:
        - ~/.npm                # npm cache directory
        - node_modules          # hoặc node_modules

  # Ví dụ cho Maven (Java)
  - restore_cache:
      keys:
        - maven-{{ checksum "pom.xml" }}
        - maven-
  - run: mvn dependency:go-offline
  - save_cache:
      key: maven-{{ checksum "pom.xml" }}
      paths:
        - ~/.m2

  # Ví dụ cho pip (Python)
  - restore_cache:
      keys:
        - pip-{{ checksum "requirements.txt" }}
        - pip-
  - run: pip install -r requirements.txt
  - save_cache:
      key: pip-{{ checksum "requirements.txt" }}
      paths:
        - ~/.cache/pip
```

**Template keys thường dùng:**
| Template | Ý Nghĩa |
|----------|---------|
| `{{ checksum "file" }}` | Hash MD5 của file |
| `{{ .Branch }}` | Tên nhánh hiện tại |
| `{{ .Revision }}` | Git commit SHA |
| `{{ epoch }}` | Unix timestamp hiện tại |
| `{{ arch }}` | CPU architecture |

### 4.4 `persist_to_workspace` và `attach_workspace`

Truyền file giữa các jobs trong cùng workflow.

```yaml
jobs:
  build:
    steps:
      - checkout
      - run: npm run build
      - persist_to_workspace:
          root: .              # Root path — đường dẫn gốc
          paths:
            - dist/            # Thư mục build output
            - package.json     # File cụ thể
            - node_modules/    # Có thể truyền cả node_modules

  deploy:
    steps:
      - attach_workspace:
          at: .                # Gắn workspace vào working directory
      - run:
          name: Deploy lên S3
          command: aws s3 sync dist/ s3://my-bucket
```

> **Workspace vs Cache:**
> - **Workspace**: Chia sẻ file **trong cùng workflow run** — không persist qua lần chạy khác
> - **Cache**: Persist **qua nhiều pipeline runs** — dùng cho dependencies

### 4.5 `store_artifacts`

Lưu file để download từ CircleCI UI.

```yaml
steps:
  - run: npm run build
  - store_artifacts:
      path: dist/              # Thư mục hoặc file cần lưu
      destination: build-output  # Tên folder trên UI (tùy chọn)
  
  # Lưu logs
  - store_artifacts:
      path: /tmp/logs/
      destination: logs
  
  # Lưu screenshots từ E2E tests — Kiểm Thử End-to-End
  - store_artifacts:
      path: cypress/screenshots/
      destination: screenshots
```

### 4.6 `store_test_results`

Lưu kết quả test theo định dạng JUnit XML — cho phép CircleCI phân tích và phát hiện flaky tests — kiểm thử không ổn định.

```yaml
steps:
  - run:
      name: Chạy tests và xuất JUnit report
      command: |
        npx jest --reporters=jest-junit \
          --outputDirectory=test-results \
          --outputName=results.xml
  
  - store_test_results:
      path: test-results/      # Thư mục chứa XML files

  # Thường kết hợp với store_artifacts để có thể download
  - store_artifacts:
      path: test-results/
      destination: test-results
```

### 4.7 `add_ssh_keys`

Thêm SSH private key — khóa SSH riêng tư vào agent, dùng để access private repos hoặc servers.

```yaml
steps:
  - add_ssh_keys:
      fingerprints:
        - "SHA256:xxxx..."     # Fingerprint của key trong CircleCI settings
  
  - run:
      name: Clone private repo
      command: git clone git@github.com:org/private-repo.git
```

### 4.8 `setup_remote_docker`

Tạo môi trường Docker riêng cho Machine executor — cho phép chạy Docker commands.

```yaml
jobs:
  docker-build:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: docker24        # Phiên bản Docker daemon
          docker_layer_caching: true  # Bật DLC để cache layers
      - run: docker build -t myapp:latest .
      - run: docker push myapp:latest
```

---

## 5. Step Điều Kiện — Conditional Steps

### 5.1 `when` và `unless`

```yaml
steps:
  - run:
      name: Deploy production
      command: ./deploy.sh production
      when: on_success           # Chạy nếu mọi step trước thành công

  - run:
      name: Gửi thông báo lỗi
      command: ./notify-failure.sh
      when: on_fail              # Chỉ chạy khi có lỗi

  - run:
      name: Cleanup
      command: rm -rf /tmp/build
      when: always               # Luôn chạy (cleanup)
```

### 5.2 Logic `when` Dựa Trên Parameters

```yaml
jobs:
  test:
    parameters:
      run-e2e:
        type: boolean
        default: false
    steps:
      - checkout
      - run: npm test

      - when:
          condition: << parameters.run-e2e >>
          steps:
            - run:
                name: Chạy E2E tests
                command: npm run test:e2e

      - unless:
          condition: << parameters.run-e2e >>
          steps:
            - run:
                name: Bỏ qua E2E tests
                command: echo "E2E tests bị tắt"
```

---

## 6. Xử Lý Lỗi Trong Steps

### 6.1 Shell Fail-fast — Dừng Ngay Khi Lỗi

CircleCI mặc định dùng shell với `-eo pipefail`:
- `-e`: Dừng script ngay khi có lệnh return non-zero exit code
- `o pipefail`: Trong pipe, nếu bất kỳ lệnh nào lỗi thì cả pipe fail

```yaml
steps:
  - run:
      shell: /bin/bash -eo pipefail  # Mặc định
      command: |
        npm install    # Nếu lỗi → dừng ở đây, không chạy tiếp
        npm test

  # Tắt fail-fast cho lệnh có thể fail mà vẫn muốn tiếp tục
  - run:
      name: Kiểm tra optional tool
      command: |
        some-optional-command || true   # || true để bỏ qua lỗi
```

### 6.2 Retry — Thử Lại

```yaml
steps:
  - run:
      name: Gọi API với retry
      command: |
        for i in 1 2 3; do
          curl -f https://api.example.com/health && break
          echo "Lần thử $i thất bại, thử lại..."
          sleep 5
        done
```

---

## 7. Artifacts và Test Results

### 7.1 Artifacts — Sản Phẩm Build

Artifacts là file output mà bạn muốn lưu lại sau khi job chạy xong (binaries, reports, logs, screenshots).

```yaml
jobs:
  build-and-test:
    steps:
      - checkout
      - run: npm run build
      - run: npm test

      # Lưu build artifacts
      - store_artifacts:
          path: dist/
          destination: production-build

      # Lưu test coverage report
      - store_artifacts:
          path: coverage/
          destination: coverage-report

      # Lưu performance report
      - store_artifacts:
          path: lighthouse-report.html
          destination: lighthouse
```

### 7.2 Test Results — Kết Quả Kiểm Thử

```yaml
steps:
  # Jest với JUnit reporter
  - run:
      name: Run Jest tests
      command: |
        npx jest \
          --reporters=default \
          --reporters=jest-junit \
          --outputFile=./test-results/jest.xml
      environment:
        JEST_JUNIT_OUTPUT_DIR: ./test-results

  # Python pytest với JUnit
  - run:
      name: Run pytest
      command: |
        pytest --junitxml=test-results/pytest.xml

  # Go tests
  - run:
      name: Run Go tests
      command: |
        go test -v ./... | go-junit-report > test-results/go.xml

  # Lưu kết quả (CircleCI phân tích XML để hiển thị test insights)
  - store_test_results:
      path: test-results/

  - store_artifacts:
      path: test-results/
```

---

## 8. Resource Class — Lớp Tài Nguyên

Resource class quyết định CPU và RAM cho mỗi job. Chọn đúng giúp tối ưu chi phí.

### 8.1 Docker Executor Resource Classes

| Resource Class | vCPU | RAM | Credits/min | Dùng Khi |
|----------------|------|-----|-------------|----------|
| `small` | 1 | 2GB | 5 | Test nhanh, script nhỏ |
| `medium` | 2 | 4GB | 10 | **Mặc định, đa dụng** |
| `medium+` | 3 | 6GB | 15 | Build vừa |
| `large` | 4 | 8GB | 20 | Build lớn, nhiều tests |
| `xlarge` | 8 | 16GB | 40 | Build nặng, ML models |
| `2xlarge` | 16 | 32GB | 80 | Compilation lớn |
| `2xlarge+` | 20 | 40GB | 100 | Monorepo builds |

### 8.2 Machine Executor Resource Classes

| Resource Class | vCPU | RAM | Credits/min |
|----------------|------|-----|-------------|
| `medium` | 2 | 7.5GB | 10 |
| `large` | 4 | 15GB | 20 |
| `xlarge` | 8 | 32GB | 40 |
| `2xlarge` | 16 | 64GB | 80 |

```yaml
jobs:
  # Job nhỏ — unit tests nhanh
  unit-test:
    docker:
      - image: cimg/node:20.0
    resource_class: small         # 1 vCPU, 2GB RAM — tiết kiệm credit
    steps:
      - checkout
      - run: npm test -- --testPathPattern="unit"

  # Job lớn — build và integration tests
  integration-test:
    docker:
      - image: cimg/node:20.0
    resource_class: large         # 4 vCPU, 8GB RAM
    parallelism: 4                # Kết hợp parallelism để nhanh hơn
    steps:
      - checkout
      - run: npm run test:integration
```

---

## 9. Ví Dụ Thực Tế

### 9.1 Pipeline Node.js Hoàn Chỉnh

```yaml
version: 2.1

executors:
  node20:
    docker:
      - image: cimg/node:20.0
    resource_class: medium
    working_directory: ~/app

jobs:
  install:
    executor: node20
    steps:
      - checkout
      - restore_cache:
          keys:
            - node-v2-{{ checksum "package-lock.json" }}
            - node-v2-
      - run:
          name: Cài đặt npm packages
          command: npm ci
      - save_cache:
          key: node-v2-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
      - persist_to_workspace:
          root: ~/app
          paths:
            - .

  lint:
    executor: node20
    steps:
      - attach_workspace:
          at: ~/app
      - run:
          name: Kiểm tra lint
          command: npm run lint

  unit-test:
    executor: node20
    parallelism: 4
    steps:
      - attach_workspace:
          at: ~/app
      - run:
          name: Chạy unit tests song song
          command: |
            circleci tests glob "src/**/*.test.ts" | \
              circleci tests split --split-by=timings | \
              xargs npx jest --forceExit \
                --reporters=default \
                --reporters=jest-junit \
                --outputFile=/tmp/test-results/jest.xml
      - store_test_results:
          path: /tmp/test-results
      - store_artifacts:
          path: /tmp/test-results
          destination: test-results

  build:
    executor: node20
    steps:
      - attach_workspace:
          at: ~/app
      - run:
          name: Build production
          command: npm run build
      - store_artifacts:
          path: dist/
          destination: dist
      - persist_to_workspace:
          root: ~/app
          paths:
            - dist/

workflows:
  ci:
    jobs:
      - install
      - lint:
          requires:
            - install
      - unit-test:
          requires:
            - install
      - build:
          requires:
            - lint
            - unit-test
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu hỏi 1: Sự khác biệt giữa `save_cache` và `persist_to_workspace`?

**Trả lời:**

| Tiêu Chí | `save_cache` | `persist_to_workspace` |
|----------|-------------|------------------------|
| Phạm vi | Nhiều pipeline runs | Chỉ trong 1 workflow run |
| Mục đích | Cache dependencies (npm, maven, pip) | Truyền build artifacts giữa jobs |
| Tốc độ | Chậm hơn (upload/download từ storage) | Nhanh hơn (shared storage) |
| Immutable | ✅ Một key chỉ save 1 lần | ❌ Overwrite được |
| Ví dụ | `~/.npm`, `~/.m2`, `vendor/` | `dist/`, `build/`, `coverage/` |

### Câu hỏi 2: Khi nào dùng Machine executor thay vì Docker executor?

**Trả lời:**

Dùng **Machine executor** khi:
1. Cần **build Docker images** (Docker-in-Docker) — Docker executor không có Docker daemon sẵn
2. Cần **privileged mode** — chạy systemd, mount filesystems
3. Test với **Docker Compose** (multi-container setup)
4. Cần truy cập `/dev/` devices hoặc kernel features đặc biệt
5. Build **iOS/macOS apps** (dùng macOS machine)

Dùng **Docker executor** khi:
- Ứng dụng thông thường (web app, API)
- Cần nhiều **service containers** (DB, Redis, RabbitMQ)
- Muốn build **nhanh hơn** (container startup nhanh hơn VM)
- Tiết kiệm **credits** (Docker rẻ hơn Machine)

### Câu hỏi 3: `when: on_fail` trong steps có tác dụng gì?

**Trả lời:**

`when: on_fail` khiến step đó chỉ chạy khi có step trước đó trong cùng job **thất bại**. Rất hữu ích cho:
- Gửi thông báo lỗi (Slack, email)
- Upload logs hoặc screenshots khi test fail
- Cleanup tài nguyên đã tạo dở

```yaml
steps:
  - run: npm test
  
  - run:
      name: Upload test failure screenshots
      command: ./upload-screenshots.sh
      when: on_fail          # Chỉ chạy khi npm test thất bại
  
  - run:
      name: Dọn dẹp môi trường test
      command: ./cleanup.sh
      when: always           # Luôn chạy để dọn dẹp
```

---

**Tiếp theo:** [3-commands.md](3-commands.md) — Commands — Lệnh Tái Sử Dụng
