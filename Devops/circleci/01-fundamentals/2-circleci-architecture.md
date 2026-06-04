# 2 — Kiến Trúc CircleCI — Pipeline → Workflow → Job → Step

> Hiểu cấu trúc phân cấp bốn tầng của CircleCI: từ Pipeline — đường ống toàn diện, qua Workflow — luồng công việc, xuống Job — công việc cụ thể, và Step — bước thực thi. Đây là nền tảng để đọc và viết mọi file cấu hình CircleCI.

---

## 📚 Mục Lục

1. [Mô Hình Phân Cấp Tổng Quan](#mô-hình-phân-cấp-tổng-quan)
2. [Pipeline — Đường Ống](#pipeline--đường-ống)
3. [Workflow — Luồng Công Việc](#workflow--luồng-công-việc)
4. [Job — Công Việc](#job--công-việc)
5. [Step — Bước Thực Thi](#step--bước-thực-thi)
6. [Vòng Đời Một Pipeline](#vòng-đời-một-pipeline)
7. [Ví Dụ Config Thực Tế](#ví-dụ-config-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Mô Hình Phân Cấp Tổng Quan

```
┌─────────────────────────────────────────────────────────┐
│  PIPELINE                                               │
│  (Toàn bộ quá trình CI/CD được kích hoạt bởi 1 sự kiện)│
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  WORKFLOW: ci-cd-pipeline                         │  │
│  │  (Nhóm các job, định nghĩa thứ tự và điều kiện)  │  │
│  │                                                   │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────────────┐ │  │
│  │  │  JOB    │──►│  JOB    │──►│      JOB        │ │  │
│  │  │  test   │   │  build  │   │     deploy      │ │  │
│  │  │         │   │         │   │                 │ │  │
│  │  │ Step 1  │   │ Step 1  │   │ Step 1          │ │  │
│  │  │ Step 2  │   │ Step 2  │   │ Step 2          │ │  │
│  │  │ Step 3  │   │ Step 3  │   │ Step 3          │ │  │
│  │  └─────────┘   └─────────┘   └─────────────────┘ │  │
│  │  (Executor A)  (Executor A)  (Executor B)         │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Quy tắc quan trọng:**
- Mỗi **Job** chạy trên một **Executor** — môi trường thực thi riêng biệt, độc lập
- Các Job trong cùng Workflow **không chia sẻ filesystem** — phải dùng Workspace để truyền artifact
- Nhiều Workflow có thể chạy song song trong một Pipeline

---

## Pipeline — Đường Ống

### Định Nghĩa

> **Pipeline** là tập hợp toàn bộ các bước tự động được kích hoạt khi có sự kiện từ VCS — Version Control System (Git repository). Một Pipeline chứa một hoặc nhiều Workflow.

### Sự Kiện Kích Hoạt Pipeline

```
┌──────────────────────────────────────────────────┐
│           Pipeline Trigger — Kích Hoạt           │
├──────────────────────────────────────────────────┤
│  git push       → Phổ biến nhất                  │
│  Pull Request   → Kiểm tra code trước merge       │
│  git tag        → Thường dùng để trigger release  │
│  Scheduled      → Cron job, chạy theo lịch        │
│  API trigger    → Gọi thủ công qua CircleCI API   │
│  Inbound webhook → Từ service bên ngoài           │
└──────────────────────────────────────────────────┘
```

### Pipeline Values — Giá Trị Pipeline Tích Hợp Sẵn

CircleCI cung cấp các biến môi trường built-in — tích hợp sẵn trong pipeline:

```yaml
# Sử dụng trong config.yml
- run:
    command: |
      echo "Pipeline ID: << pipeline.id >>"
      echo "Git revision: << pipeline.git.revision >>"
      echo "Branch: << pipeline.git.branch >>"
      echo "Triggered by: << pipeline.trigger_source >>"
```

| Biến | Giá Trị Ví Dụ | Dùng Để |
| ---- | ------------- | ------- |
| `pipeline.id` | `550e8400-e29b-41d4-a716` | Tracking, logging |
| `pipeline.number` | `42` | Hiển thị số pipeline |
| `pipeline.git.branch` | `feature/login` | Điều kiện filter |
| `pipeline.git.revision` | `a5d4c3b` | Tag Docker image |
| `pipeline.git.tag` | `v1.2.0` | Release logic |
| `pipeline.trigger_source` | `webhook` / `scheduled_pipeline` | Điều kiện chạy |

---

## Workflow — Luồng Công Việc

### Định Nghĩa

> **Workflow** định nghĩa tập hợp các Job, thứ tự thực thi, và các điều kiện (filter nhánh, approval). Một Pipeline có thể có nhiều Workflow chạy song song.

### Các Kiểu Workflow

#### 1. Sequential Workflow — Luồng Tuần Tự

```yaml
workflows:
  sequential:
    jobs:
      - test
      - build:
          requires:
            - test      # build chỉ chạy khi test pass
      - deploy:
          requires:
            - build     # deploy chỉ chạy khi build xong
```

```
test ──► build ──► deploy
```

#### 2. Parallel Workflow — Luồng Song Song

```yaml
workflows:
  parallel:
    jobs:
      - test-unit       # Chạy song song
      - test-lint       # Chạy song song
      - test-security   # Chạy song song
      - build:
          requires:
            - test-unit
            - test-lint
            - test-security   # Chờ cả 3 xong rồi mới build
```

```
test-unit ──┐
test-lint ──┼──► build
test-security ──┘
```

#### 3. Approval Workflow — Luồng Cần Duyệt

```yaml
workflows:
  with-approval:
    jobs:
      - build
      - approve-production:
          type: approval        # Dừng và chờ người duyệt click
          requires:
            - build
          filters:
            branches:
              only: main
      - deploy-production:
          requires:
            - approve-production
```

#### 4. Scheduled Workflow — Luồng Theo Lịch

```yaml
workflows:
  nightly-build:
    triggers:
      - schedule:
          cron: "0 2 * * *"    # 2:00 AM UTC mỗi ngày
          filters:
            branches:
              only: main
    jobs:
      - full-test-suite
      - security-scan
```

### Filters — Bộ Lọc Workflow

```yaml
workflows:
  feature-ci:
    jobs:
      - test:
          filters:
            branches:
              ignore: main     # Không chạy trên nhánh main

  release:
    jobs:
      - deploy:
          filters:
            branches:
              only: main       # Chỉ chạy trên main
            tags:
              only: /^v.*/     # Chỉ chạy khi tag bắt đầu bằng "v"
```

---

## Job — Công Việc

### Định Nghĩa

> **Job** là đơn vị công việc trong CircleCI. Mỗi Job chạy trong một môi trường cô lập (Docker container hoặc VM — Virtual Machine) và chứa danh sách các Step.

### Cấu Trúc Job

```yaml
jobs:
  build-and-test:               # Tên job — đặt có ý nghĩa
    docker:                     # Executor — môi trường chạy
      - image: cimg/node:20.0
    working_directory: ~/app    # Thư mục làm việc
    resource_class: medium      # CPU/RAM (2 vCPU, 4 GB)
    environment:                # Biến môi trường cấp job
      NODE_ENV: test
    parallelism: 4              # Chạy song song 4 container
    steps:
      - checkout
      - run: npm ci
      - run: npm test
```

### Thuộc Tính Job Quan Trọng

| Thuộc Tính | Mô Tả | Ví Dụ |
| ---------- | ----- | ----- |
| `docker` / `machine` / `macos` | Executor type | `docker: - image: cimg/node:20.0` |
| `resource_class` | CPU/RAM | `medium`, `large`, `xlarge` |
| `working_directory` | Thư mục làm việc | `~/project` |
| `parallelism` | Số container song song | `4` |
| `environment` | Biến môi trường cố định | `NODE_ENV: test` |
| `shell` | Shell mặc định | `/bin/bash -eo pipefail` |

### Job Isolation — Sự Cô Lập Job

```
Job A (test)          Job B (build)
┌─────────────┐       ┌─────────────┐
│  Container  │       │  Container  │
│  /home/app  │       │  /home/app  │  ← Hoàn toàn khác nhau
│  (fresh)    │       │  (fresh)    │
└─────────────┘       └─────────────┘
       │                     │
       │ Không chia sẻ filesystem
       │
       └─► Dùng Workspace để truyền file giữa jobs
```

---

## Step — Bước Thực Thi

### Định Nghĩa

> **Step** là đơn vị nhỏ nhất trong CircleCI — một lệnh cụ thể được thực thi trong Job. Các Step chạy tuần tự, mặc định dừng lại nếu Step trước thất bại.

### Built-in Steps — Bước Tích Hợp Sẵn

#### `checkout` — Lấy Code

```yaml
steps:
  - checkout    # Lấy code từ VCS về working_directory
                # Tự động cấu hình SSH key
```

#### `run` — Chạy Lệnh Shell

```yaml
steps:
  - run: npm test              # Dạng ngắn

  - run:                       # Dạng đầy đủ
      name: Chạy kiểm thử
      command: |
        npm ci
        npm test
      environment:
        NODE_ENV: test
      no_output_timeout: 10m   # Timeout nếu không có output sau 10 phút
      when: always             # Luôn chạy, kể cả step trước fail
```

#### `save_cache` / `restore_cache` — Lưu / Khôi Phục Cache

```yaml
steps:
  - restore_cache:
      keys:
        - node-v1-{{ checksum "package-lock.json" }}
        - node-v1-          # Fallback key — khóa dự phòng

  - run: npm ci

  - save_cache:
      key: node-v1-{{ checksum "package-lock.json" }}
      paths:
        - ~/.npm
        - node_modules
```

#### `persist_to_workspace` / `attach_workspace` — Truyền File Giữa Jobs

```yaml
# Job build — lưu artifact vào workspace
- persist_to_workspace:
    root: .           # Thư mục gốc của workspace
    paths:
      - dist/         # Chỉ lưu thư mục dist
      - package.json

# Job deploy — nhận artifact từ workspace
- attach_workspace:
    at: .             # Gắn workspace vào thư mục hiện tại
```

#### `store_artifacts` — Lưu Artifact Để Xem Sau

```yaml
- store_artifacts:
    path: test-results/
    destination: test-results    # Tên folder trong CircleCI UI

- store_artifacts:
    path: coverage/
    destination: coverage-report
```

#### `store_test_results` — Lưu Kết Quả Test

```yaml
- store_test_results:
    path: test-results/junit.xml   # File XML theo JUnit format
    # → CircleCI tự hiển thị test stats, phát hiện flaky tests
```

#### `add_ssh_keys` — Thêm SSH Key

```yaml
- add_ssh_keys:
    fingerprints:
      - "SO:ME:FIN:G:ER:PR:IN:T"  # Fingerprint của key trong CircleCI settings
```

### Step Điều Kiện — Conditional Steps

```yaml
steps:
  - run:
      name: Deploy chỉ trên main
      command: ./deploy.sh
      when: on_success    # Mặc định — chỉ chạy khi mọi thứ OK

  - run:
      name: Cleanup khi fail
      command: ./cleanup.sh
      when: on_fail       # Chỉ chạy khi có step thất bại

  - run:
      name: Luôn notify
      command: ./notify.sh
      when: always        # Luôn chạy bất kể kết quả
```

---

## Vòng Đời Một Pipeline

### Từ Push Đến Kết Quả

```
1. Developer: git push origin feature/login
                    │
                    ▼
2. GitHub webhook → CircleCI API
   { "ref": "feature/login", "sha": "abc123", ... }
                    │
                    ▼
3. CircleCI đọc .circleci/config.yml
   - Parse YAML
   - Xác định workflows phù hợp với branch filters
   - Lên lịch các jobs
                    │
                    ▼
4. CircleCI Executor provisioning — Cấp phát môi trường
   - Pull Docker image (hoặc spin up VM)
   - Mount workspace
   - Inject environment variables
                    │
                    ▼
5. Steps thực thi tuần tự trong Job
   - checkout → install → lint → test → build
                    │
                    ▼
6. Job kết thúc → CircleCI thu hồi môi trường
   - Container bị xóa
   - Workspace (nếu persist) được lưu tạm thời
                    │
                    ▼
7. Workflow tiếp tục với job tiếp theo (nếu pass)
   hoặc dừng và thông báo (nếu fail)
                    │
                    ▼
8. Pipeline hoàn thành → Trạng thái gửi về GitHub
   ✅ green check mark hoặc ❌ red X trên PR
```

### Trạng Thái Pipeline

```
queued    → Đang chờ runner — máy chạy khả dụng
running   → Đang thực thi
success   → Tất cả jobs pass ✅
failed    → Ít nhất 1 job fail ❌
canceled  → Người dùng hoặc auto-cancel hủy
on_hold   → Đang chờ approval
```

---

## Ví Dụ Config Thực Tế

### Pipeline Node.js Đầy Đủ

```yaml
version: 2.1

# Executors — Môi Trường Tái Sử Dụng
executors:
  node-executor:
    docker:
      - image: cimg/node:20.0
        auth:
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_TOKEN
    resource_class: medium
    working_directory: ~/app

# Commands — Lệnh Tái Sử Dụng
commands:
  install-deps:
    steps:
      - restore_cache:
          keys:
            - npm-v2-{{ checksum "package-lock.json" }}
            - npm-v2-
      - run: npm ci
      - save_cache:
          key: npm-v2-{{ checksum "package-lock.json" }}
          paths: [~/.npm]

# Jobs — Công Việc
jobs:
  test:
    executor: node-executor
    parallelism: 2        # Chia test thành 2 luồng song song
    steps:
      - checkout
      - install-deps
      - run:
          name: Chạy tests song song
          command: |
            circleci tests glob "src/**/*.test.js" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --forceExit
      - store_test_results:
          path: test-results/

  build:
    executor: node-executor
    steps:
      - checkout
      - install-deps
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/, package.json, package-lock.json]

  deploy-staging:
    executor: node-executor
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy lên staging
          command: ./scripts/deploy.sh staging

  approve-production:
    # Job đặc biệt type: approval — không cần executor
    type: approval

  deploy-production:
    executor: node-executor
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy lên production
          command: ./scripts/deploy.sh production

# Workflows — Luồng Công Việc
workflows:
  ci-cd:
    jobs:
      # CI Phase — Giai Đoạn CI: chạy trên mọi nhánh
      - test
      - build:
          requires: [test]

      # Staging Deploy: chỉ trên main
      - deploy-staging:
          requires: [build]
          filters:
            branches:
              only: main

      # Production: cần duyệt thủ công
      - approve-production:
          type: approval
          requires: [deploy-staging]
          filters:
            branches:
              only: main

      - deploy-production:
          requires: [approve-production]
          filters:
            branches:
              only: main
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Giải thích sự khác biệt giữa Pipeline, Workflow, Job và Step

**Trả lời mẫu:**

> **Pipeline** là toàn bộ quá trình CI/CD được kích hoạt bởi một sự kiện (git push, webhook). Một pipeline chứa một hoặc nhiều Workflow.
>
> **Workflow** định nghĩa nhóm các Job và quan hệ giữa chúng — thứ tự chạy, điều kiện chạy, branch filters. Nhiều Workflow trong một pipeline có thể chạy song song.
>
> **Job** là đơn vị công việc, chạy trong một môi trường cô lập (Docker container hoặc VM). Mỗi Job chứa danh sách các Step và không chia sẻ filesystem với Job khác.
>
> **Step** là lệnh cụ thể được thực thi trong Job, ví dụ `checkout`, `run`, `save_cache`. Các Step chạy tuần tự trong một Job.

### Câu 2: Tại sao các Job không chia sẻ filesystem? Làm thế nào để truyền file?

**Trả lời mẫu:**

> Mỗi Job chạy trong container hoặc VM riêng biệt, được tạo mới từ đầu để đảm bảo môi trường sạch và reproducible — có thể tái tạo. Việc cô lập này tránh tình trạng side effects giữa các job.
>
> Để truyền file giữa các Job, dùng **Workspace**: job A dùng `persist_to_workspace` để lưu file vào vùng lưu trữ tạm thời, job B dùng `attach_workspace` để nhận file đó. Workspace tồn tại trong phạm vi một Workflow, không qua Pipeline khác.

### Câu 3: Khi nào dùng `when: always` cho một Step?

**Trả lời mẫu:**

> Dùng `when: always` cho các Step cần chạy bất kể kết quả của Pipeline, ví dụ: gửi thông báo Slack khi build fail, cleanup tài nguyên tạm thời, hay lưu test logs để debug. Mặc định, Step sẽ bị skip nếu Step trước đó fail, nhưng `when: always` đảm bảo Step luôn được thực thi.

---

## 🔗 Đọc Tiếp

- [3-executors.md](3-executors.md) — Chọn Executor phù hợp
- [../02-configuration/2-jobs-and-steps.md](../02-configuration/2-jobs-and-steps.md) — Cấu hình Jobs và Steps chi tiết
- [../03-workflows/1-sequential-workflow.md](../03-workflows/1-sequential-workflow.md) — Workflow nâng cao

---

**Thời Gian Đọc:** 45–60 phút  
**Cập Nhật:** 2026-05-18
