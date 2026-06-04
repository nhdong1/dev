# 🔁 Commands — Lệnh Tái Sử Dụng Trong CircleCI

> Commands — Lệnh Tái Sử Dụng cho phép nhóm nhiều steps thành một đơn vị có tên, có thể gọi lại trong nhiều jobs. Đây là cơ chế DRY (Don't Repeat Yourself — Không Lặp Lại Bản Thân) của CircleCI.

---

## 📋 Mục Lục

1. [Commands Là Gì?](#1-commands-là-gì)
2. [Cú Pháp Khai Báo Command](#2-cú-pháp-khai-báo-command)
3. [Parameters Trong Commands](#3-parameters-trong-commands)
4. [Commands Lồng Nhau](#4-commands-lồng-nhau)
5. [Built-in Commands vs Custom Commands](#5-built-in-commands-vs-custom-commands)
6. [Ví Dụ Thực Tế](#6-ví-dụ-thực-tế)
7. [Commands vs Orbs](#7-commands-vs-orbs)
8. [Best Practices](#8-best-practices)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Commands Là Gì?

**Command** là một tập hợp các **steps** được đặt tên và có thể:
- Gọi lại trong nhiều jobs trong cùng file config
- Nhận **parameters — tham số** để linh hoạt hóa
- Chứa bất kỳ built-in steps hoặc custom commands khác

**Tương tự như:** Hàm (function) hoặc phương thức (method) trong lập trình.

```
Không có commands (lặp code):          Với commands (DRY):
──────────────────────────────         ──────────────────────────
job: build                             commands:
  steps:                                 install-deps:
    - restore_cache: ...                   steps:
    - run: npm ci                            - restore_cache: ...
    - save_cache: ...                        - run: npm ci
                                             - save_cache: ...
job: test
  steps:                                 jobs:
    - restore_cache: ...  ← lặp          build:
    - run: npm ci         ← lặp            steps:
    - save_cache: ...     ← lặp              - install-deps   ← gọn
                                         test:
job: lint                                  steps:
  steps:                                     - install-deps   ← gọn
    - restore_cache: ...  ← lặp
    - run: npm ci         ← lặp
    - save_cache: ...     ← lặp
```

---

## 2. Cú Pháp Khai Báo Command

```yaml
version: 2.1

commands:
  # Tên command (dùng kebab-case — gạch nối)
  install-node-deps:
    # Mô tả command (hiển thị trong UI và tài liệu)
    description: "Cài đặt Node.js dependencies với cache"
    
    # Parameters tùy chọn
    parameters:
      cache-version:
        type: string
        default: "v1"
        description: "Phiên bản cache key để invalidate cache thủ công"
    
    # Steps của command
    steps:
      - restore_cache:
          keys:
            - node-<< parameters.cache-version >>-{{ checksum "package-lock.json" }}
            - node-<< parameters.cache-version >>-
      - run:
          name: Cài đặt npm packages
          command: npm ci
      - save_cache:
          key: node-<< parameters.cache-version >>-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - install-node-deps              # Gọi command không có args
      - run: npm run build

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - install-node-deps:             # Gọi command với args
          cache-version: "v2"          # Override default parameter
      - run: npm test
```

---

## 3. Parameters Trong Commands

### 3.1 Các Kiểu Parameter

```yaml
commands:
  flexible-deploy:
    description: "Deploy linh hoạt với nhiều tùy chọn"
    parameters:
      # String — Chuỗi
      environment:
        type: string
        default: "staging"
        description: "Môi trường deploy: staging hoặc production"
      
      # Boolean — Logic
      dry-run:
        type: boolean
        default: false
        description: "Nếu true, chỉ mô phỏng, không deploy thực"
      
      # Integer — Số nguyên
      timeout:
        type: integer
        default: 300
        description: "Timeout tính bằng giây"
      
      # Enum — Giá trị liệt kê
      cloud-provider:
        type: enum
        enum: [aws, gcp, azure]
        default: aws
        description: "Cloud provider mục tiêu"
      
      # Env var name — Tên biến môi trường
      aws-access-key:
        type: env_var_name
        default: AWS_ACCESS_KEY_ID
        description: "Tên biến môi trường chứa AWS access key"
      
      # Steps — Các bước tùy chỉnh
      pre-deploy-steps:
        type: steps
        default: []
        description: "Steps thêm trước khi deploy"
    
    steps:
      # Dùng parameters trong steps
      - run:
          name: Chuẩn bị deploy
          command: |
            echo "Deploy lên: << parameters.environment >>"
            echo "Cloud: << parameters.cloud-provider >>"
            echo "AWS Key var: $<< parameters.aws-access-key >>"
      
      # Điều kiện dựa trên boolean parameter
      - when:
          condition: << parameters.dry-run >>
          steps:
            - run: echo "DRY RUN — Không deploy thực"
      
      - unless:
          condition: << parameters.dry-run >>
          steps:
            # Chạy pre-deploy-steps được truyền vào
            - steps: << parameters.pre-deploy-steps >>
            - run:
                name: Deploy thực
                command: ./deploy.sh << parameters.environment >>
                no_output_timeout: << parameters.timeout >>s
```

### 3.2 Cách Dùng `env_var_name` Parameter

```yaml
commands:
  aws-login:
    parameters:
      access-key:
        type: env_var_name
        default: AWS_ACCESS_KEY_ID
      secret-key:
        type: env_var_name
        default: AWS_SECRET_ACCESS_KEY
    steps:
      - run:
          name: Cấu hình AWS CLI
          command: |
            aws configure set aws_access_key_id $<< parameters.access-key >>
            aws configure set aws_secret_access_key $<< parameters.secret-key >>
            aws configure set region us-east-1

jobs:
  deploy-prod:
    steps:
      - aws-login:
          access-key: PROD_AWS_ACCESS_KEY_ID   # Tên biến env cho prod
          secret-key: PROD_AWS_SECRET_KEY

  deploy-staging:
    steps:
      - aws-login                              # Dùng default values
```

### 3.3 `steps` Parameter — Tham Số Kiểu Steps

`steps` parameter cho phép inject thêm steps vào command — cực kỳ linh hoạt.

```yaml
commands:
  with-database:
    description: "Chạy steps với database context"
    parameters:
      steps:
        type: steps
        description: "Steps cần chạy sau khi DB sẵn sàng"
    steps:
      - run:
          name: Chờ database
          command: |
            dockerize -wait tcp://localhost:5432 -timeout 60s
      - run:
          name: Chạy migrations
          command: npm run db:migrate
      
      # Chèn steps được truyền vào
      - steps: << parameters.steps >>
      
      - run:
          name: Cleanup DB
          command: npm run db:clean
          when: always

jobs:
  integration-test:
    docker:
      - image: cimg/node:20.0
      - image: cimg/postgres:14.0
    steps:
      - checkout
      - with-database:
          steps:
            - run: npm run test:integration
            - store_test_results:
                path: test-results/
```

---

## 4. Commands Lồng Nhau

Command có thể gọi command khác. Tuy nhiên, tránh lồng quá nhiều cấp để dễ đọc.

```yaml
commands:
  # Command cơ bản
  git-setup:
    steps:
      - checkout
      - run:
          name: Cấu hình Git user
          command: |
            git config user.email "ci@example.com"
            git config user.name "CircleCI Bot"

  # Command dùng command khác
  install-and-build:
    parameters:
      node-version:
        type: string
        default: "20"
    steps:
      - git-setup                      # Gọi command khác
      - restore_cache:
          keys:
            - node-<< parameters.node-version >>-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: node-<< parameters.node-version >>-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
      - run: npm run build

  # Command cấp cao hơn
  full-ci-setup:
    steps:
      - install-and-build              # Gọi install-and-build
      - run: npm test

jobs:
  ci:
    docker:
      - image: cimg/node:20.0
    steps:
      - full-ci-setup                  # Gọi command cấp cao nhất
```

---

## 5. Built-in Commands vs Custom Commands

| Loại | Ví Dụ | Tính Năng |
|------|-------|-----------|
| **Built-in** | `checkout`, `run`, `save_cache`, `restore_cache` | Tích hợp sẵn trong CircleCI |
| **Custom** | `install-deps`, `deploy-to-s3` | Do bạn định nghĩa trong `commands:` |
| **Orb Commands** | `node/install-packages`, `aws-cli/setup` | Do orb cung cấp, dùng như custom |

---

## 6. Ví Dụ Thực Tế

### 6.1 Command Cài Đặt Và Cache Nhiều Loại Ngôn Ngữ

```yaml
commands:
  # Python
  install-python-deps:
    parameters:
      venv-path:
        type: string
        default: "venv"
    steps:
      - restore_cache:
          keys:
            - pip-v1-{{ checksum "requirements.txt" }}
            - pip-v1-
      - run:
          name: Tạo virtual environment và cài đặt
          command: |
            python -m venv << parameters.venv-path >>
            source << parameters.venv-path >>/bin/activate
            pip install -r requirements.txt
      - save_cache:
          key: pip-v1-{{ checksum "requirements.txt" }}
          paths:
            - ~/.cache/pip

  # Go
  install-go-deps:
    steps:
      - restore_cache:
          keys:
            - go-mod-v1-{{ checksum "go.sum" }}
            - go-mod-v1-
      - run: go mod download
      - save_cache:
          key: go-mod-v1-{{ checksum "go.sum" }}
          paths:
            - /go/pkg/mod

  # Ruby/Bundler
  install-ruby-gems:
    steps:
      - restore_cache:
          keys:
            - gems-v1-{{ checksum "Gemfile.lock" }}
            - gems-v1-
      - run: bundle install --path vendor/bundle
      - save_cache:
          key: gems-v1-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle
```

### 6.2 Command Deploy Đa Môi Trường

```yaml
commands:
  deploy-kubernetes:
    description: "Deploy ứng dụng lên Kubernetes cluster"
    parameters:
      cluster:
        type: string
        description: "Tên EKS cluster"
      namespace:
        type: string
        default: "default"
      image-tag:
        type: string
        default: "$CIRCLE_SHA1"
      dry-run:
        type: boolean
        default: false
    steps:
      - run:
          name: Cấu hình kubectl
          command: |
            aws eks update-kubeconfig \
              --region us-east-1 \
              --name << parameters.cluster >>
      - run:
          name: Cập nhật image tag
          command: |
            kubectl set image deployment/myapp \
              myapp=myapp:<< parameters.image-tag >> \
              -n << parameters.namespace >> \
              << # parameters.dry-run | ternary "--dry-run=client" "" >>
      - unless:
          condition: << parameters.dry-run >>
          steps:
            - run:
                name: Chờ rollout hoàn thành
                command: |
                  kubectl rollout status deployment/myapp \
                    -n << parameters.namespace >> \
                    --timeout=5m

jobs:
  deploy-staging:
    steps:
      - deploy-kubernetes:
          cluster: my-staging-cluster
          namespace: staging
          image-tag: "$CIRCLE_SHA1"

  deploy-production:
    steps:
      - deploy-kubernetes:
          cluster: my-prod-cluster
          namespace: production
          image-tag: "$CIRCLE_TAG"
```

### 6.3 Command Thông Báo Slack

```yaml
commands:
  notify-slack:
    description: "Gửi thông báo Slack về trạng thái deploy"
    parameters:
      status:
        type: enum
        enum: [success, failure, started]
        description: "Trạng thái cần thông báo"
      channel:
        type: string
        default: "#deployments"
      message:
        type: string
        default: ""
    steps:
      - run:
          name: Gửi thông báo Slack
          when: always
          command: |
            STATUS="<< parameters.status >>"
            EMOJI=""
            COLOR=""
            case $STATUS in
              "success") EMOJI="✅"; COLOR="good" ;;
              "failure") EMOJI="❌"; COLOR="danger" ;;
              "started") EMOJI="🚀"; COLOR="#439FE0" ;;
            esac
            
            curl -X POST $SLACK_WEBHOOK_URL \
              -H "Content-type: application/json" \
              -d "{
                \"channel\": \"<< parameters.channel >>\",
                \"attachments\": [{
                  \"color\": \"$COLOR\",
                  \"text\": \"$EMOJI Deploy $STATUS: $CIRCLE_PROJECT_REPONAME @ $CIRCLE_BRANCH\"
                }]
              }"
```

---

## 7. Commands vs Orbs

| Tiêu Chí | Custom Commands | Orb Commands |
|----------|----------------|-------------|
| Phạm vi | Chỉ trong 1 file config | Dùng được ở mọi project |
| Chia sẻ | Không (trừ khi copy) | Publish lên registry, share toàn tổ chức |
| Cập nhật | Sửa từng file | Cập nhật 1 lần, tất cả dùng ngay |
| Phức tạp | Đơn giản | Phức tạp hơn (orb structure) |
| Dùng khi | Logic đặc thù của project | Logic chung, tái dùng nhiều project |

---

## 8. Best Practices

### 8.1 Đặt Tên Rõ Ràng

```yaml
# ❌ Tên mơ hồ
commands:
  setup:        # Setup gì?
    steps: ...
  run-stuff:    # Chạy gì?
    steps: ...

# ✅ Tên mô tả
commands:
  install-node-deps:
    steps: ...
  build-docker-image:
    steps: ...
  deploy-to-s3:
    steps: ...
```

### 8.2 Luôn Thêm `description`

```yaml
commands:
  install-node-deps:
    description: |
      Cài đặt Node.js dependencies từ package-lock.json.
      Tự động restore/save cache theo checksum của package-lock.json.
    parameters:
      cache-key-prefix:
        type: string
        default: "node-v1"
        description: "Prefix của cache key. Tăng số version để invalidate cache."
    steps:
      - restore_cache: ...
```

### 8.3 Xử Lý Cleanup Với `when: always`

```yaml
commands:
  run-with-cleanup:
    parameters:
      steps:
        type: steps
    steps:
      - run: ./setup-resources.sh

      - steps: << parameters.steps >>

      - run:
          name: Dọn dẹp tài nguyên
          command: ./cleanup-resources.sh
          when: always    # Đảm bảo luôn dọn dẹp dù steps fail
```

### 8.4 Tránh Side Effects Ẩn

```yaml
# ❌ Command có side effect ẩn — thay đổi environment không rõ ràng
commands:
  setup-env:
    steps:
      - run: export NODE_ENV=production  # Export này KHÔNG persist sang steps sau!

# ✅ Dùng environment key rõ ràng
commands:
  setup-env:
    steps:
      - run:
          command: echo "Thiết lập môi trường"
          environment:
            NODE_ENV: production   # Chỉ áp dụng cho run step này
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu hỏi 1: Commands trong CircleCI là gì và khi nào dùng?

**Trả lời:**

Commands là các **tập steps được đặt tên** khai báo trong block `commands:`. Dùng commands khi:
1. **Cùng một nhóm steps lặp lại** ở nhiều jobs (cài đặt dependencies, cleanup)
2. **Logic phức tạp** cần tách ra để dễ test và maintain
3. **Muốn tham số hóa** cấu hình để linh hoạt (môi trường staging/prod, timeout)

Commands chỉ có phạm vi trong một file config. Nếu cần chia sẻ giữa nhiều dự án, nên tạo **Orb**.

---

### Câu hỏi 2: Sự khác biệt giữa command `steps` parameter và YAML anchors?

**Trả lời:**

| | `steps` Parameter | YAML Anchors (`&` / `*`) |
|--|-------------------|--------------------------|
| Tính năng | CircleCI-native, hỗ trợ parameters | YAML thuần túy |
| Validate | CircleCI validate đúng cú pháp | Không validate CircleCI-specific |
| Linh hoạt | Có thể inject steps động theo điều kiện | Chỉ copy nguyên khối |
| Readable | Tên có ý nghĩa | Cần biết anchor được khai báo ở đâu |
| Khuyến nghị | ✅ Ưu tiên | ⚠️ Dùng cho trường hợp đơn giản |

---

### Câu hỏi 3: Tại sao `export VAR=value` trong một `run` step không ảnh hưởng đến steps tiếp theo?

**Trả lời:**

Mỗi `run` step khởi chạy một **process mới** (shell mới). Environment variables được set bằng `export` chỉ tồn tại trong process đó. Khi step kết thúc, process chết và biến đó mất.

**Giải pháp:**
1. Dùng `environment:` key trong `run` step
2. Viết vào `$BASH_ENV` — CircleCI tự source file này trước mỗi step:
   ```bash
   echo "export MY_VAR=value" >> $BASH_ENV
   ```
3. Dùng job-level `environment:` cho biến dùng nhiều steps

---

**Tiếp theo:** [4-parameters.md](4-parameters.md) — Parameters — Tham Số Pipeline
