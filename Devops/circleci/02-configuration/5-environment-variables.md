# 🔐 Environment Variables — Biến Môi Trường Trong CircleCI

> Environment Variables — Biến Môi Trường là cách truyền cấu hình và secrets — bí mật vào pipeline mà không hardcode vào source code. CircleCI có hệ thống phân cấp env vars nhiều lớp từ built-in đến org-level.

---

## 📋 Mục Lục

1. [Phân Cấp Environment Variables](#1-phân-cấp-environment-variables)
2. [Built-in Environment Variables](#2-built-in-environment-variables)
3. [Project-Level Variables — Biến Cấp Dự Án](#3-project-level-variables--biến-cấp-dự-án)
4. [Context Variables — Biến Cấp Ngữ Cảnh](#4-context-variables--biến-cấp-ngữ-cảnh)
5. [Config-Level Variables — Biến Trong Config](#5-config-level-variables--biến-trong-config)
6. [Thứ Tự Ưu Tiên — Override Order](#6-thứ-tự-ưu-tiên--override-order)
7. [Dùng Variables Trong Config](#7-dùng-variables-trong-config)
8. [Best Practices Bảo Mật](#8-best-practices-bảo-mật)
9. [Ví Dụ Thực Tế](#9-ví-dụ-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Phân Cấp Environment Variables

```
Cấp độ cao nhất (ưu tiên cao nhất)
┌─────────────────────────────────────────────────┐
│ 4. Shell export trong run step                  │ ← Chỉ trong step đó
│    run: export MY_VAR=value                     │
├─────────────────────────────────────────────────┤
│ 3. Config-level (trong config.yml)              │ ← Job/step scope
│    environment: MY_VAR: value                   │
├─────────────────────────────────────────────────┤
│ 2. Project-level Variables                      │ ← Một project
│    CircleCI UI → Project Settings → Env Vars   │
├─────────────────────────────────────────────────┤
│ 1. Context Variables                            │ ← Nhiều projects
│    Org-level, có access control                 │
├─────────────────────────────────────────────────┤
│ 0. Built-in CircleCI Variables                  │ ← Tự động inject
│    CIRCLE_BRANCH, CIRCLE_SHA1, etc.             │
└─────────────────────────────────────────────────┘
Cấp độ thấp nhất (ưu tiên thấp nhất)
```

---

## 2. Built-in Environment Variables

CircleCI tự động inject các biến này vào mọi job. Không cần khai báo.

### 2.1 Thông Tin Pipeline — Pipeline Information

| Biến | Mô Tả | Ví Dụ |
|------|--------|-------|
| `CIRCLE_PIPELINE_ID` | ID duy nhất của pipeline | `5034460f-c7a3-4c6e-8d6b-...` |
| `CIRCLE_PIPELINE_NUMBER` | Số thứ tự pipeline trong project | `123` |
| `CIRCLE_WORKFLOW_ID` | ID duy nhất của workflow | `12a3b4cd-...` |
| `CIRCLE_WORKFLOW_JOB_ID` | ID của job trong workflow | `abc123...` |

### 2.2 Thông Tin Build — Build Information

| Biến | Mô Tả | Ví Dụ |
|------|--------|-------|
| `CIRCLE_BUILD_NUM` | Số thứ tự build | `456` |
| `CIRCLE_BUILD_URL` | URL của build hiện tại | `https://circleci.com/...` |
| `CIRCLE_JOB` | Tên job đang chạy | `build`, `test`, `deploy` |
| `CIRCLE_NODE_INDEX` | Index của node song song (0-based) | `0`, `1`, `2`, `3` |
| `CIRCLE_NODE_TOTAL` | Tổng số nodes song song | `4` |

### 2.3 Thông Tin Git — Git Information

| Biến | Mô Tả | Ví Dụ |
|------|--------|-------|
| `CIRCLE_SHA1` | Git commit SHA đầy đủ | `a91a4f92...` |
| `CIRCLE_BRANCH` | Tên nhánh hiện tại | `main`, `feature/login` |
| `CIRCLE_TAG` | Tên tag (nếu được trigger bởi tag) | `v1.2.3` |
| `CIRCLE_PR_NUMBER` | Số Pull Request (nếu là PR build) | `42` |
| `CIRCLE_PR_REPONAME` | Tên repo của PR gốc | `my-repo` |
| `CIRCLE_COMPARE_URL` | URL so sánh commits | `https://github.com/...` |

### 2.4 Thông Tin Repository — Repository Information

| Biến | Mô Tả | Ví Dụ |
|------|--------|-------|
| `CIRCLE_PROJECT_REPONAME` | Tên repository | `my-app` |
| `CIRCLE_PROJECT_USERNAME` | Tên org/user | `my-org` |
| `CIRCLE_REPOSITORY_URL` | URL của repo | `https://github.com/my-org/my-app` |

### 2.5 Thông Tin Người Dùng — User Information

| Biến | Mô Tả | Ví Dụ |
|------|--------|-------|
| `CIRCLE_USERNAME` | Username của người trigger build | `john-doe` |

### 2.6 Dùng Built-in Variables

```yaml
jobs:
  build-and-tag:
    steps:
      - checkout
      - run:
          name: Build Docker image với SHA tag
          command: |
            # Tag image với commit SHA để có thể trace nguồn gốc
            docker build \
              -t myapp:$CIRCLE_SHA1 \
              -t myapp:latest \
              --label git-commit=$CIRCLE_SHA1 \
              --label build-url=$CIRCLE_BUILD_URL \
              .
      
      - run:
          name: Thông tin build
          command: |
            echo "Branch: $CIRCLE_BRANCH"
            echo "Tag: ${CIRCLE_TAG:-không có tag}"
            echo "Build #: $CIRCLE_BUILD_NUM"
            echo "Committer: $CIRCLE_USERNAME"
            
            # Điều kiện dựa trên branch
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              echo "Đây là build trên main — sẽ deploy production"
            fi
```

---

## 3. Project-Level Variables — Biến Cấp Dự Án

**Cách thiết lập:** CircleCI Dashboard → Project → Project Settings → Environment Variables

### 3.1 Đặc Điểm

- **Chỉ dùng trong một project**
- Được mã hóa (encrypted) và không bao giờ hiển thị lại sau khi lưu
- Inject vào tất cả jobs của project (trừ khi bị override)
- Phù hợp cho: Database credentials, API keys của project, deployment secrets

### 3.2 Khi Nào Dùng Project-Level Variables

```
✅ Phù hợp:
  - Secrets đặc thù của một project
  - Database connection strings
  - Service account keys riêng của project

❌ Không phù hợp:
  - Secrets dùng chung nhiều projects → Dùng Context
  - Thông tin không nhạy cảm → Khai báo trong config.yml
  - Secrets cần access control → Dùng Context với security groups
```

### 3.3 Quản Lý Qua API

```bash
# Xem danh sách env vars (chỉ thấy tên, không thấy giá trị)
curl --request GET \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/envvar" \
  --header "Circle-Token: $CIRCLECI_TOKEN"

# Thêm env var mới
curl --request POST \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/envvar" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"name": "MY_SECRET", "value": "secret-value"}'

# Xóa env var
curl --request DELETE \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/envvar/MY_SECRET" \
  --header "Circle-Token: $CIRCLECI_TOKEN"
```

---

## 4. Context Variables — Biến Cấp Ngữ Cảnh

**Context — Ngữ Cảnh** là nhóm environment variables được quản lý ở cấp **organization** (tổ chức), có thể chia sẻ cho nhiều projects.

**Cách thiết lập:** CircleCI Dashboard → Organization Settings → Contexts

### 4.1 Đặc Điểm

- Được quản lý bởi **Org Admin**
- Có thể gán cho **nhiều projects** — không cần duplicate
- Hỗ trợ **Security Groups** — Nhóm Bảo Mật (chỉ một số người/teams có quyền dùng)
- Tự động **masked** trong logs — không hiển thị giá trị
- Phiên bản **restricted context** cho môi trường production

### 4.2 Dùng Context Trong Workflow

```yaml
workflows:
  deploy:
    jobs:
      # Job không cần secrets đặc biệt
      - test
      
      # Job deploy staging — dùng context staging
      - deploy-staging:
          requires:
            - test
          context:
            - org-global          # Context chứa secrets dùng chung
            - aws-staging         # Context riêng cho AWS staging
      
      # Job deploy production — context production (restricted)
      - deploy-production:
          requires:
            - deploy-staging
          context:
            - org-global
            - aws-production      # Context restricted — chỉ team leads có quyền
          filters:
            branches:
              only: main
```

### 4.3 Multiple Contexts

Một job có thể dùng nhiều contexts. Nếu có xung đột tên biến, context được khai báo sau sẽ override context trước.

```yaml
workflows:
  ci:
    jobs:
      - build:
          context:
            - base-context         # AWS_REGION, SLACK_WEBHOOK
            - docker-context       # DOCKERHUB_USER, DOCKERHUB_TOKEN
            - deploy-context       # AWS_ACCESS_KEY_ID (override base nếu trùng tên)
```

### 4.4 Context Trong API — Restricted Context

```yaml
# Restricted context — chỉ áp dụng khi:
# 1. Job chạy từ branch/tag phù hợp
# 2. User có trong security group của context
# Nếu không đủ quyền → job bị skip, không fail

workflows:
  deploy-prod:
    jobs:
      - deploy:
          context:
            - production-secrets   # Restricted context
          filters:
            branches:
              only: main           # Thêm branch filter để bảo mật hơn
```

---

## 5. Config-Level Variables — Biến Trong Config

### 5.1 Job-Level Environment Variables

```yaml
jobs:
  build:
    docker:
      - image: cimg/node:20.0
    
    # Biến môi trường ở cấp job — dùng cho tất cả steps trong job này
    environment:
      NODE_ENV: production
      BUILD_DIR: ./dist
      LOG_LEVEL: info
    
    steps:
      - checkout
      - run: echo $NODE_ENV    # production
      - run: echo $BUILD_DIR   # ./dist
```

### 5.2 Step-Level Environment Variables

```yaml
jobs:
  test:
    steps:
      # Biến chỉ áp dụng trong run step này
      - run:
          name: Unit tests với custom env
          command: npm test
          environment:
            NODE_ENV: test
            TEST_TIMEOUT: "30000"
            JEST_WORKERS: "2"
      
      # Run step tiếp theo không có NODE_ENV=test
      - run: echo "NODE_ENV bây giờ là: ${NODE_ENV:-undefined}"
```

### 5.3 Docker Image Environment Variables

```yaml
jobs:
  test-with-db:
    docker:
      - image: cimg/node:20.0
        # Biến cho container chính
        environment:
          DATABASE_URL: "postgresql://testuser:testpass@localhost:5432/testdb"
      
      - image: cimg/postgres:14.0
        # Biến khởi tạo PostgreSQL
        environment:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
      
      - image: redis:7-alpine
        # Redis không cần biến đặc biệt
```

### 5.4 Sử Dụng `$BASH_ENV` Để Share Variables Giữa Steps

Mỗi `run` step chạy trong một shell mới — `export` trong step này không sang step khác. Dùng `$BASH_ENV` để giải quyết.

```yaml
steps:
  - run:
      name: Tính toán version
      command: |
        # Tính version từ git tags
        APP_VERSION=$(git describe --tags --always --dirty)
        BUILD_DATE=$(date +%Y%m%d-%H%M%S)
        
        # Ghi vào $BASH_ENV — CircleCI source file này trước mỗi step
        echo "export APP_VERSION=${APP_VERSION}" >> $BASH_ENV
        echo "export BUILD_DATE=${BUILD_DATE}" >> $BASH_ENV
        echo "export IMAGE_TAG=${APP_VERSION}-${BUILD_DATE}" >> $BASH_ENV
  
  - run:
      name: Build Docker image
      command: |
        # Biến từ step trước giờ đây có sẵn
        echo "Building image: myapp:${IMAGE_TAG}"
        docker build -t myapp:${IMAGE_TAG} .
  
  - run:
      name: Push Docker image
      command: |
        docker push myapp:${IMAGE_TAG}
        echo "Pushed: myapp:${IMAGE_TAG}"
```

---

## 6. Thứ Tự Ưu Tiên — Override Order

Khi cùng tên biến xuất hiện ở nhiều cấp:

```
Cao nhất (override tất cả)
    │
    ▼
 Step-level environment   (environment: trong run step)
    │
    ▼
 Job-level environment    (environment: trong job)
    │
    ▼
 Project-level variables  (Project Settings UI)
    │
    ▼
 Context variables        (Organization contexts)
    │
    ▼
 Built-in variables       (CIRCLE_BRANCH, v.v.)
    │
    ▼
Thấp nhất (bị override bởi tất cả)
```

**Ví dụ thực tế:**

```yaml
# Context "prod-context" có: API_URL=https://prod.api.com
# Project settings có: API_URL=https://project.api.com
# Job environment có: API_URL=https://job.api.com

jobs:
  test:
    environment:
      API_URL: https://job.api.com     # Step và job-level win
    steps:
      - run:
          environment:
            API_URL: https://step.api.com  # Step-level WINS — giá trị này được dùng
          command: echo $API_URL       # Prints: https://step.api.com
```

---

## 7. Dùng Variables Trong Config

### 7.1 Trong Shell Commands

```yaml
steps:
  - run:
      command: |
        # Truy cập trực tiếp
        echo "Branch: $CIRCLE_BRANCH"
        
        # Dùng giá trị mặc định nếu biến chưa set
        REGION=${AWS_REGION:-us-east-1}
        
        # Check biến có tồn tại không
        if [ -z "${DATABASE_URL:-}" ]; then
          echo "LỖI: DATABASE_URL chưa được set!"
          exit 1
        fi
        
        # Dùng trong URL
        curl "https://api.example.com/deploy?env=$DEPLOY_ENV&version=$APP_VERSION"
```

### 7.2 Interpolation Trong YAML — Nội Suy Trong YAML

**Quan trọng:** YAML không tự động expand environment variables. Phải dùng `run:` step để expand.

```yaml
# ❌ SAI — YAML không expand $VARIABLE
jobs:
  build:
    docker:
      - image: cimg/node:$NODE_VERSION   # Không hoạt động!

# ✅ Đúng — dùng giá trị tĩnh
jobs:
  build:
    docker:
      - image: cimg/node:20.0

# ✅ Đúng — dùng parameters nếu cần dynamic
jobs:
  build:
    parameters:
      node-version:
        type: string
        default: "20.0"
    docker:
      - image: cimg/node:<< parameters.node-version >>
```

### 7.3 Sử Dụng Trong `context` và `filters` — Không Được Phép

Context và filters không hỗ trợ variable interpolation — phải dùng giá trị tĩnh.

```yaml
# ❌ Không hoạt động
workflows:
  deploy:
    jobs:
      - deploy:
          context: $CONTEXT_NAME   # Không expand!

# ✅ Phải dùng tên tĩnh
workflows:
  deploy:
    jobs:
      - deploy:
          context: production-context
```

---

## 8. Best Practices Bảo Mật

### 8.1 Nguyên Tắc Least Privilege — Đặc Quyền Tối Thiểu

```yaml
# ❌ Dùng context với nhiều quyền cho tất cả jobs
workflows:
  ci:
    jobs:
      - test:
          context: production-full-access   # Test không cần quyền production!
      - deploy:
          context: production-full-access

# ✅ Chỉ inject secrets khi cần
workflows:
  ci:
    jobs:
      - test                               # Không cần context
      - deploy:
          context: production-deploy-only  # Context chỉ có quyền deploy
          requires:
            - test
```

### 8.2 Không Hardcode Secrets

```yaml
# ❌ TUYỆT ĐỐI KHÔNG LÀM THẾ NÀY
jobs:
  deploy:
    environment:
      AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE    # Hardcoded!
      DATABASE_URL: postgresql://admin:password123@prod-db:5432/app
    steps:
      - run: aws s3 sync dist/ s3://my-bucket

# ✅ Dùng context hoặc project variables
jobs:
  deploy:
    steps:
      - run:
          name: Deploy
          command: |
            # AWS_ACCESS_KEY_ID và DATABASE_URL được inject từ context
            aws s3 sync dist/ s3://my-bucket
```

### 8.3 Masked Variables — Biến Được Che Giấu

CircleCI tự động mask — che giấu giá trị của secrets trong logs. Nhưng cần lưu ý:

```bash
# ❌ Có thể leak secret qua base64 encoding
echo $SECRET | base64  # CircleCI KHÔNG mask giá trị encoded

# ❌ Có thể leak qua error messages
node -e "require('$SECRET')"  # Error message có thể chứa giá trị

# ✅ Cẩn thận khi debug
set +x                # Tắt command echo trong bash
echo "Length: ${#SECRET}"  # Chỉ log độ dài, không log giá trị
```

### 8.4 Rotation Secrets — Xoay Vòng Bí Mật

```bash
# Script rotation secrets không downtime
# 1. Thêm NEW_AWS_KEY song song với OLD_AWS_KEY
# 2. Update code dùng NEW_AWS_KEY
# 3. Verify jobs chạy đúng với NEW_AWS_KEY  
# 4. Xóa OLD_AWS_KEY
# Không bao giờ có thời điểm secret bị thiếu

# Thêm secret mới qua API
curl --request POST \
  --url "https://circleci.com/api/v2/project/github/ORG/REPO/envvar" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --data '{"name": "NEW_AWS_KEY", "value": "new-key-value"}'
```

---

## 9. Ví Dụ Thực Tế

### 9.1 Multi-Environment Config Dùng Variables

```yaml
version: 2.1

jobs:
  deploy:
    docker:
      - image: cimg/base:stable
    parameters:
      environment:
        type: enum
        enum: [staging, production]
    steps:
      - run:
          name: Deploy lên << parameters.environment >>
          command: |
            case "<< parameters.environment >>" in
              "staging")
                DEPLOY_URL="$STAGING_URL"
                DEPLOY_KEY="$STAGING_DEPLOY_KEY"
                ;;
              "production")
                DEPLOY_URL="$PRODUCTION_URL"
                DEPLOY_KEY="$PRODUCTION_DEPLOY_KEY"
                ;;
            esac
            
            echo "Deploying to: $DEPLOY_URL"
            curl -X POST "$DEPLOY_URL/deploy" \
              -H "Authorization: Bearer $DEPLOY_KEY" \
              -d "{\"version\": \"$CIRCLE_SHA1\"}"

workflows:
  deploy-pipeline:
    jobs:
      - deploy:
          name: deploy-staging
          environment: staging
          context:
            - staging-secrets     # Chứa STAGING_URL, STAGING_DEPLOY_KEY
          filters:
            branches:
              only: develop
      
      - deploy:
          name: deploy-production
          environment: production
          context:
            - production-secrets  # Chứa PRODUCTION_URL, PRODUCTION_DEPLOY_KEY
          filters:
            branches:
              only: main
```

### 9.2 Kiểm Tra Required Variables

```yaml
commands:
  verify-env-vars:
    description: "Kiểm tra các biến môi trường bắt buộc"
    parameters:
      required-vars:
        type: string
        description: "Danh sách biến cần kiểm tra, phân cách bởi space"
    steps:
      - run:
          name: Kiểm tra required environment variables
          command: |
            MISSING_VARS=""
            for VAR in << parameters.required-vars >>; do
              if [ -z "${!VAR:-}" ]; then
                MISSING_VARS="$MISSING_VARS $VAR"
              fi
            done
            
            if [ -n "$MISSING_VARS" ]; then
              echo "❌ Thiếu các biến môi trường bắt buộc:$MISSING_VARS"
              echo "Hãy thiết lập trong Project Settings hoặc Context"
              exit 1
            else
              echo "✅ Tất cả biến môi trường bắt buộc đã được thiết lập"
            fi

jobs:
  deploy-production:
    steps:
      - verify-env-vars:
          required-vars: "AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY DATABASE_URL SLACK_WEBHOOK"
      - run: ./deploy.sh
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu hỏi 1: Sự khác biệt giữa Project Variables và Contexts?

**Trả lời:**

| | Project Variables | Contexts |
|--|-------------------|---------|
| Phạm vi | Một project | Nhiều projects (org-level) |
| Quản lý bởi | Project admin | Org admin |
| Access control | Không có | Có (Security Groups) |
| Chia sẻ | Không thể | Chia sẻ giữa teams/projects |
| Dùng khi | Secrets riêng của project | Credentials dùng chung |

**Ví dụ thực tế:** Công ty có 10 microservices đều cần AWS credentials. Thay vì set `AWS_ACCESS_KEY_ID` ở 10 projects, tạo 1 context `aws-deploy` và inject cho tất cả.

---

### Câu hỏi 2: Tại sao `export VAR=value` trong một run step không hoạt động ở step tiếp theo?

**Trả lời:**

Mỗi `run` step mở một **shell process mới**. `export` chỉ tồn tại trong process hiện tại. Khi step kết thúc, process chết và biến mất.

**Giải pháp:** Ghi vào `$BASH_ENV` — CircleCI source file này trước mỗi step:
```bash
echo "export MY_VAR=value" >> $BASH_ENV
```

---

### Câu hỏi 3: Làm thế nào để bảo mật secrets tốt nhất trong CircleCI?

**Trả lời:**

1. **Dùng Contexts** với Security Groups — không cho phép jobs không liên quan truy cập production secrets
2. **Không bao giờ hardcode** secrets trong config.yml hoặc script files
3. **Restrict contexts** với branch filters — chỉ trigger từ branches tin cậy
4. **Monitor Audit Log** — Nhật Ký Kiểm Toán để phát hiện truy cập bất thường
5. **Rotate secrets định kỳ** — đặc biệt sau khi offboard nhân viên
6. **Dùng OIDC** — OpenID Connect thay vì long-lived credentials cho cloud providers
7. **Principle of Least Privilege** — mỗi job chỉ có secrets cần thiết, không hơn

---

**Module Hoàn Thành!**  
Tiếp theo: [03-workflows/README.md](../03-workflows/README.md) — Workflow Design — Thiết Kế Luồng Công Việc
