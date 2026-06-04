# 🎛️ Parameters — Tham Số Trong CircleCI

> Parameters — Tham Số cho phép tạo pipeline, jobs và commands linh hoạt bằng cách truyền giá trị vào lúc chạy. Thay vì viết nhiều configs giống nhau, một config có parameters có thể phục vụ nhiều mục đích khác nhau.

---

## 📋 Mục Lục

1. [Parameters Là Gì?](#1-parameters-là-gì)
2. [Các Kiểu Dữ Liệu Parameter](#2-các-kiểu-dữ-liệu-parameter)
3. [Pipeline Parameters](#3-pipeline-parameters)
4. [Job Parameters](#4-job-parameters)
5. [Command Parameters](#5-command-parameters)
6. [Executor Parameters](#6-executor-parameters)
7. [Kích Hoạt Pipeline Với Parameters Qua API](#7-kích-hoạt-pipeline-với-parameters-qua-api)
8. [Parameters Trong Dynamic Config](#8-parameters-trong-dynamic-config)
9. [Ví Dụ Thực Tế](#9-ví-dụ-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Parameters Là Gì?

Parameters là **biến đầu vào** được khai báo và truyền vào các thành phần của config. Chúng hoạt động ở 4 cấp độ:

```
Pipeline Parameters  → Giá trị truyền vào toàn bộ pipeline (từ API, trigger)
    ↓
Job Parameters       → Giá trị truyền vào một job cụ thể (từ workflow)
    ↓
Command Parameters   → Giá trị truyền vào một command (từ job steps)
    ↓
Executor Parameters  → Giá trị truyền vào executor (Docker image version, etc.)
```

**Cú pháp tham chiếu:**
```yaml
<< parameters.param-name >>   # Tham chiếu parameter trong cùng scope
<< pipeline.parameters.name >> # Tham chiếu pipeline parameter từ bất kỳ đâu
```

---

## 2. Các Kiểu Dữ Liệu Parameter

### 2.1 Tổng Quan Các Kiểu

| Kiểu | Mô Tả | Ví Dụ |
|------|--------|-------|
| `string` | Chuỗi ký tự | `"staging"`, `"us-east-1"` |
| `boolean` | Đúng/sai | `true`, `false` |
| `integer` | Số nguyên | `4`, `300`, `8080` |
| `enum` | Giá trị từ tập cố định | `aws`, `gcp`, `azure` |
| `env_var_name` | Tên biến môi trường (không phải giá trị) | `AWS_ACCESS_KEY_ID` |
| `steps` | Danh sách steps | `[- run: echo hi]` |
| `executor` | Tên executor | `node-executor` |

### 2.2 `string` — Chuỗi

```yaml
parameters:
  deploy-environment:
    type: string
    default: "staging"
    description: "Tên môi trường deploy"
```

### 2.3 `boolean` — Logic

```yaml
parameters:
  run-performance-tests:
    type: boolean
    default: false
    description: "Có chạy performance tests không"
```

### 2.4 `integer` — Số Nguyên

```yaml
parameters:
  parallelism-level:
    type: integer
    default: 4
    description: "Số luồng song song cho test"
```

### 2.5 `enum` — Giá Trị Liệt Kê

```yaml
parameters:
  cloud-provider:
    type: enum
    enum: [aws, gcp, azure, do]
    default: aws
    description: "Cloud provider target"
  
  node-version:
    type: enum
    enum: ["18", "20", "22"]
    default: "20"
```

### 2.6 `env_var_name` — Tên Biến Môi Trường

Thay vì truyền giá trị secret trực tiếp, truyền **tên** của biến môi trường chứa secret đó.

```yaml
parameters:
  api-key-var:
    type: env_var_name
    default: DEFAULT_API_KEY
    description: "Tên biến môi trường chứa API key"

steps:
  - run:
      name: Gọi API
      command: |
        # Dùng giá trị của biến môi trường có tên được truyền vào
        curl -H "Authorization: Bearer $<< parameters.api-key-var >>" \
          https://api.example.com/data
```

### 2.7 `steps` — Danh Sách Steps

```yaml
parameters:
  post-deploy-steps:
    type: steps
    default: []
    description: "Steps chạy sau khi deploy"

steps:
  - run: ./deploy.sh
  - steps: << parameters.post-deploy-steps >>  # Inject steps vào đây
```

---

## 3. Pipeline Parameters

Pipeline Parameters được khai báo ở top-level và có thể được truyền khi trigger pipeline qua API.

### 3.1 Khai Báo Pipeline Parameters

```yaml
version: 2.1

# Khai báo tại top level
parameters:
  # Môi trường deploy
  deploy-environment:
    type: enum
    enum: [development, staging, production]
    default: staging
  
  # Phiên bản ứng dụng
  app-version:
    type: string
    default: ""
  
  # Có chạy integration tests không
  run-integration-tests:
    type: boolean
    default: true
  
  # Số luồng song song
  test-parallelism:
    type: integer
    default: 4

jobs:
  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Deploy với parameters
          command: |
            echo "Environment: << pipeline.parameters.deploy-environment >>"
            echo "Version: << pipeline.parameters.app-version >>"
            ./deploy.sh \
              --env << pipeline.parameters.deploy-environment >> \
              --version << pipeline.parameters.app-version >>

  test:
    docker:
      - image: cimg/node:20.0
    parallelism: << pipeline.parameters.test-parallelism >>
    steps:
      - checkout
      - run: npm test

workflows:
  main:
    jobs:
      - test
      - deploy:
          requires:
            - test
          filters:
            branches:
              only: main
      
      # Chỉ chạy integration tests nếu parameter = true
      - integration-test:
          requires:
            - test
          # Dùng when condition dựa trên pipeline parameter
```

### 3.2 Dùng `when` Với Pipeline Parameters Để Điều Kiện Workflow

```yaml
version: 2.1

parameters:
  run-nightly-build:
    type: boolean
    default: false
  
  run-deploy:
    type: boolean
    default: true

workflows:
  # Workflow thông thường — chạy khi run-nightly-build = false
  ci:
    when:
      not: << pipeline.parameters.run-nightly-build >>
    jobs:
      - build
      - test

  # Workflow nightly — chỉ chạy khi được trigger với run-nightly-build = true
  nightly-full-test:
    when: << pipeline.parameters.run-nightly-build >>
    jobs:
      - build
      - unit-test
      - integration-test
      - performance-test
      - e2e-test

  # Workflow deploy — chỉ chạy khi run-deploy = true
  deploy-workflow:
    when: << pipeline.parameters.run-deploy >>
    jobs:
      - deploy
```

---

## 4. Job Parameters

Job parameters cho phép gọi cùng một job nhiều lần trong workflow với các giá trị khác nhau.

### 4.1 Khai Báo Job Parameters

```yaml
jobs:
  deploy:
    parameters:
      environment:
        type: string
        description: "Môi trường deploy"
      aws-region:
        type: string
        default: "ap-southeast-1"
      enable-canary:
        type: boolean
        default: false
    
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Deploy lên << parameters.environment >>
          command: |
            aws configure set region << parameters.aws-region >>
            
            if << parameters.enable-canary >>; then
              ./deploy-canary.sh << parameters.environment >>
            else
              ./deploy-full.sh << parameters.environment >>
            fi
```

### 4.2 Gọi Job Với Parameters Trong Workflow

```yaml
workflows:
  deploy-pipeline:
    jobs:
      - build
      
      # Deploy staging — không cần canary
      - deploy:
          name: deploy-staging             # Đặt tên riêng khi gọi cùng job nhiều lần
          requires:
            - build
          environment: staging
          aws-region: ap-southeast-1
          filters:
            branches:
              only: develop
      
      # Deploy production — dùng canary
      - deploy:
          name: deploy-production
          requires:
            - build
          environment: production
          aws-region: ap-southeast-1
          enable-canary: true
          filters:
            branches:
              only: main
```

### 4.3 Matrix — Ma Trận Job

Matrix là cách đặc biệt để chạy cùng một job nhiều lần với các combinations — tổ hợp khác nhau.

```yaml
jobs:
  test-compatibility:
    parameters:
      node-version:
        type: string
      os:
        type: enum
        enum: [linux, windows]
    
    docker:
      - image: cimg/node:<< parameters.node-version >>
    steps:
      - checkout
      - run: npm test

workflows:
  compatibility-matrix:
    jobs:
      - test-compatibility:
          matrix:
            parameters:
              node-version: ["18", "20", "22"]
              os: [linux]
          # Tạo ra 3 jobs:
          # test-compatibility-18-linux
          # test-compatibility-20-linux
          # test-compatibility-22-linux
      
      - deploy:
          requires:
            # Matrix expansion: requires tất cả combinations
            - test-compatibility
```

---

## 5. Command Parameters

Xem chi tiết tại [3-commands.md](3-commands.md). Tóm tắt các kiểu parameter:

```yaml
commands:
  example-command:
    parameters:
      text:
        type: string
        default: "Hello"
      
      verbose:
        type: boolean
        default: false
      
      retries:
        type: integer
        default: 3
      
      log-level:
        type: enum
        enum: [debug, info, warn, error]
        default: info
      
      secret-var:
        type: env_var_name
        default: MY_SECRET
      
      extra-steps:
        type: steps
        default: []
    
    steps:
      - run:
          name: Main step
          command: |
            echo "<< parameters.text >>"
            echo "Log level: << parameters.log-level >>"
            echo "Secret from: $<< parameters.secret-var >>"
      
      - when:
          condition: << parameters.verbose >>
          steps:
            - run: echo "Verbose mode bật"
      
      - steps: << parameters.extra-steps >>
```

---

## 6. Executor Parameters

```yaml
executors:
  node-executor:
    parameters:
      node-version:
        type: string
        default: "20"
      resource:
        type: enum
        enum: [small, medium, large]
        default: medium
    docker:
      - image: cimg/node:<< parameters.node-version >>
    resource_class: << parameters.resource >>

jobs:
  build-node18:
    executor:
      name: node-executor
      node-version: "18"
      resource: small
    steps:
      - checkout
      - run: npm test

  build-node22:
    executor:
      name: node-executor
      node-version: "22"
      resource: large        # Build lớn cần nhiều tài nguyên hơn
    steps:
      - checkout
      - run: npm test
```

---

## 7. Kích Hoạt Pipeline Với Parameters Qua API

### 7.1 REST API Trigger — Kích Hoạt Qua API

```bash
# Trigger pipeline với custom parameters
curl --request POST \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/pipeline" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "branch": "main",
    "parameters": {
      "deploy-environment": "production",
      "app-version": "2.5.0",
      "run-integration-tests": true,
      "test-parallelism": 8
    }
  }'

# Trigger nightly build
curl --request POST \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/pipeline" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "branch": "main",
    "parameters": {
      "run-nightly-build": true
    }
  }'
```

### 7.2 GitHub Actions Trigger CircleCI

```yaml
# .github/workflows/trigger-circleci.yml
name: Trigger CircleCI Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Deploy environment"
        required: true
        type: choice
        options: [staging, production]
      version:
        description: "App version"
        required: true

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger CircleCI Pipeline
        run: |
          curl --request POST \
            --url "https://circleci.com/api/v2/project/github/${{ github.repository }}/pipeline" \
            --header "Circle-Token: ${{ secrets.CIRCLECI_TOKEN }}" \
            --header "Content-Type: application/json" \
            --data '{
              "branch": "main",
              "parameters": {
                "deploy-environment": "${{ inputs.environment }}",
                "app-version": "${{ inputs.version }}"
              }
            }'
```

---

## 8. Parameters Trong Dynamic Config

**Dynamic Config — Cấu Hình Động** dùng pipeline parameters để truyền thông tin từ setup config sang continuation config.

```yaml
# .circleci/config.yml (setup config)
version: 2.1
setup: true

orbs:
  continuation: circleci/continuation@1.0.0

jobs:
  detect-changes:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Phát hiện thay đổi
          command: |
            # Kiểm tra file thay đổi
            CHANGED=$(git diff --name-only HEAD~1 HEAD)
            
            # Quyết định build gì
            if echo "$CHANGED" | grep -q "^frontend/"; then
              echo "export BUILD_FRONTEND=true" >> $BASH_ENV
            fi
            if echo "$CHANGED" | grep -q "^backend/"; then
              echo "export BUILD_BACKEND=true" >> $BASH_ENV
            fi
      
      - continuation/continue:
          configuration_path: .circleci/continue.yml
          parameters: |
            {
              "build-frontend": ${BUILD_FRONTEND:-false},
              "build-backend": ${BUILD_BACKEND:-false}
            }

workflows:
  setup:
    jobs:
      - detect-changes
```

```yaml
# .circleci/continue.yml (continuation config)
version: 2.1

parameters:
  build-frontend:
    type: boolean
    default: false
  build-backend:
    type: boolean
    default: false

workflows:
  conditional-build:
    jobs:
      - build-frontend:
          filters:
            # Chỉ chạy nếu parameter = true
      - build-backend:
          filters:
            #

# Dùng when condition tốt hơn
  smart-build:
    when:
      or:
        - << pipeline.parameters.build-frontend >>
        - << pipeline.parameters.build-backend >>
    jobs:
      - build-frontend:
          # using when inline
      - build-backend:
          #
```

---

## 9. Ví Dụ Thực Tế

### 9.1 Multi-Environment Deployment Pipeline

```yaml
version: 2.1

parameters:
  target-env:
    type: enum
    enum: [dev, staging, prod]
    default: dev
  image-tag:
    type: string
    default: "latest"
  skip-tests:
    type: boolean
    default: false
  notify-slack:
    type: boolean
    default: true

executors:
  deploy-executor:
    parameters:
      env:
        type: string
    docker:
      - image: cimg/base:stable
    environment:
      DEPLOY_ENV: << parameters.env >>

commands:
  run-if-not-skipped:
    parameters:
      steps:
        type: steps
      skip:
        type: boolean
        default: false
    steps:
      - unless:
          condition: << parameters.skip >>
          steps: << parameters.steps >>

jobs:
  unit-test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run-if-not-skipped:
          skip: << pipeline.parameters.skip-tests >>
          steps:
            - run: npm test

  deploy:
    parameters:
      environment:
        type: string
    executor:
      name: deploy-executor
      env: << parameters.environment >>
    steps:
      - run:
          name: Deploy << parameters.environment >>
          command: |
            ./scripts/deploy.sh \
              --env << parameters.environment >> \
              --tag << pipeline.parameters.image-tag >>
      - when:
          condition: << pipeline.parameters.notify-slack >>
          steps:
            - run:
                name: Thông báo Slack
                command: |
                  curl -X POST $SLACK_WEBHOOK \
                    -d "{\"text\": \"Deploy << parameters.environment >> hoàn thành: << pipeline.parameters.image-tag >>\"}"

workflows:
  deploy-pipeline:
    jobs:
      - unit-test
      - deploy:
          name: deploy-<< pipeline.parameters.target-env >>
          requires:
            - unit-test
          environment: << pipeline.parameters.target-env >>
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu hỏi 1: Pipeline parameters khác job parameters như thế nào?

**Trả lời:**

| | Pipeline Parameters | Job Parameters |
|--|---------------------|----------------|
| Scope | Toàn bộ pipeline | Chỉ một job |
| Khai báo | Top-level `parameters:` | Trong `jobs.<name>.parameters:` |
| Truyền giá trị | Từ API trigger, scheduled pipeline | Từ workflow `jobs:` section |
| Tham chiếu | `<< pipeline.parameters.name >>` | `<< parameters.name >>` |
| Dùng khi | Feature flags, environment selection | Job reuse với config khác nhau |

---

### Câu hỏi 2: Tại sao dùng `env_var_name` parameter thay vì `string` cho secrets?

**Trả lời:**

`env_var_name` parameter nhận **tên** của biến môi trường, không phải giá trị. Lợi ích:

1. **Bảo mật:** Giá trị secret không bao giờ xuất hiện trong config file (không bị commit lên repo)
2. **Linh hoạt:** Các môi trường khác nhau có thể dùng biến môi trường khác nhau
3. **Tái sử dụng:** Command có thể dùng với nhiều secrets khác nhau

```yaml
# Sai (không bao giờ làm thế này)
commands:
  login:
    parameters:
      api-key:
        type: string     # Giá trị có thể bị log ra
    steps:
      - run: curl -H "Authorization: << parameters.api-key >>" ...

# Đúng
commands:
  login:
    parameters:
      api-key-var:
        type: env_var_name   # Tên biến, không phải giá trị
        default: API_KEY
    steps:
      - run: curl -H "Authorization: $<< parameters.api-key-var >>" ...
```

---

### Câu hỏi 3: Làm thế nào để một workflow chỉ chạy khi được trigger theo cách cụ thể?

**Trả lời:**

Dùng pipeline parameters kết hợp `when` condition trong workflow:

```yaml
parameters:
  run-nightly:
    type: boolean
    default: false

workflows:
  # Workflow bình thường — không chạy khi nightly trigger
  ci:
    when:
      not: << pipeline.parameters.run-nightly >>
    jobs:
      - test

  # Workflow nightly — chỉ chạy khi trigger với run-nightly: true
  nightly:
    when: << pipeline.parameters.run-nightly >>
    jobs:
      - full-test-suite
      - performance-test
```

Trigger nightly bằng cron scheduled pipeline hoặc API call với parameter.

---

**Tiếp theo:** [5-environment-variables.md](5-environment-variables.md) — Environment Variables — Biến Môi Trường
