# 4. Custom Orb Development — Phát Triển Orb Tùy Chỉnh

> Hướng dẫn toàn diện về cách thiết kế, viết code, kiểm thử và publish orb của riêng bạn lên CircleCI Orb Registry.

---

## 📚 Mục Lục

1. [Tại Sao Cần Custom Orb](#tại-sao-cần-custom-orb)
2. [Cấu Trúc Thư Mục Orb](#cấu-trúc-thư-mục-orb)
3. [Thiết Kế Orb Schema](#thiết-kế-orb-schema)
4. [Viết Commands](#viết-commands)
5. [Viết Jobs](#viết-jobs)
6. [Viết Executors](#viết-executors)
7. [Parameters Nâng Cao](#parameters-nâng-cao)
8. [Kiểm Thử Orb](#kiểm-thử-orb)
9. [Publish Orb Lên Registry](#publish-orb-lên-registry)
10. [Orb Development Pipeline](#orb-development-pipeline)
11. [Best Practices](#best-practices)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Custom Orb

### Khi Nào Cần Tạo Custom Orb

```
Tình huống thực tế:
1. Công ty có 20+ repositories, tất cả deploy lên cùng một hệ thống nội bộ
2. Mỗi team tự viết cấu hình deploy → không nhất quán, khó maintain
3. Giải pháp: tạo internal orb chuẩn hóa quy trình deploy

Lợi ích:
✅ Viết một lần — dùng ở mọi nơi trong tổ chức
✅ Update một chỗ — áp dụng cho tất cả projects
✅ Best practices được enforce tự động
✅ Giảm thời gian onboarding cho team mới
```

### Custom Orb vs Inline Orb

| | Custom Orb | Inline Orb |
|--|-----------|-----------|
| **Phạm vi** | Nhiều repositories | Một config file |
| **Versioning** | Có — semantic versioning | Không |
| **Publish** | Có (public hoặc private) | Không |
| **Testing** | Cần pipeline riêng | Đơn giản hơn |
| **Độ phức tạp** | Cao hơn | Thấp hơn |

---

## Cấu Trúc Thư Mục Orb

### Structure Được Khuyến Nghị

```
my-company-orb/
├── .circleci/
│   └── config.yml              Pipeline để test và publish orb
├── src/
│   ├── @orb.yml                File metadata của orb (mô tả, display info)
│   ├── commands/
│   │   ├── setup.yml           Command setup-environment
│   │   ├── deploy.yml          Command deploy-app
│   │   └── notify.yml          Command send-notification
│   ├── jobs/
│   │   ├── test.yml            Job chạy tests
│   │   └── build-deploy.yml    Job build và deploy
│   ├── executors/
│   │   ├── default.yml         Executor mặc định
│   │   └── with-db.yml         Executor có database service
│   └── examples/
│       ├── basic-usage.yml     Ví dụ sử dụng cơ bản
│       └── advanced-usage.yml  Ví dụ nâng cao
├── tests/
│   └── integration/
│       └── test-deploy.yml     Config để kiểm thử orb
└── README.md
```

### Cách Orb Packing Hoạt Động

CircleCI CLI có lệnh `orb pack` để gộp tất cả file YAML thành một file orb duy nhất:

```bash
# Gộp src/ thành orb.yml
circleci orb pack src/ > orb.yml

# Validate orb
circleci orb validate orb.yml

# Publish orb
circleci orb publish orb.yml my-namespace/my-orb@dev:alpha
```

---

## Thiết Kế Orb Schema

### File `src/@orb.yml` — Metadata

```yaml
# src/@orb.yml
version: 2.1

description: >
  Orb chuẩn hóa quy trình deploy ứng dụng lên Kubernetes
  tại My Company. Hỗ trợ EKS (AWS), GKE (GCP) và AKS (Azure).

display:
  home_url: https://github.com/my-company/circleci-orb
  source_url: https://github.com/my-company/circleci-orb/tree/main/src
```

---

## Viết Commands

### Command Với Parameters Đầy Đủ

```yaml
# src/commands/deploy.yml
description: >
  Deploy ứng dụng lên Kubernetes cluster với zero-downtime rolling update.

parameters:
  # Tham số bắt buộc
  cluster-name:
    type: string
    description: Tên Kubernetes cluster cần deploy tới

  # Tham số có giá trị mặc định
  namespace:
    type: string
    default: "default"
    description: Kubernetes namespace để deploy

  # Tham số kiểu enum — chỉ chấp nhận giá trị cụ thể
  strategy:
    type: enum
    enum: ["rolling", "recreate", "blue-green"]
    default: "rolling"
    description: Chiến lược deploy

  # Tham số kiểu boolean
  wait-for-completion:
    type: boolean
    default: true
    description: Chờ cho rollout hoàn thành trước khi tiếp tục

  # Tham số kiểu integer
  timeout-minutes:
    type: integer
    default: 10
    description: Thời gian chờ tối đa (phút)

steps:
  - run:
      name: "Kiểm tra cluster: << parameters.cluster-name >>"
      command: |
        kubectl cluster-info \
          --context << parameters.cluster-name >> \
          --namespace << parameters.namespace >>

  - run:
      name: "Deploy với chiến lược: << parameters.strategy >>"
      command: |
        STRATEGY="<< parameters.strategy >>"

        if [ "$STRATEGY" = "rolling" ]; then
          kubectl set image deployment/$APP_NAME \
            app=$DOCKER_IMAGE:$CIRCLE_SHA1 \
            --namespace << parameters.namespace >>

        elif [ "$STRATEGY" = "recreate" ]; then
          kubectl rollout restart deployment/$APP_NAME \
            --namespace << parameters.namespace >>

        elif [ "$STRATEGY" = "blue-green" ]; then
          # Blue-Green deployment logic
          ./scripts/blue-green-deploy.sh \
            << parameters.namespace >> \
            $CIRCLE_SHA1
        fi

  - when:
      condition: << parameters.wait-for-completion >>
      steps:
        - run:
            name: Chờ rollout hoàn thành
            command: |
              kubectl rollout status deployment/$APP_NAME \
                --namespace << parameters.namespace >> \
                --timeout << parameters.timeout-minutes >>m
```

### Command Với Conditional Steps — Bước Có Điều Kiện

```yaml
# src/commands/setup.yml
description: Thiết lập môi trường deploy

parameters:
  cloud-provider:
    type: enum
    enum: ["aws", "gcp", "azure"]
    description: Cloud provider để cấu hình credentials

  install-kubectl:
    type: boolean
    default: true
    description: Có cài kubectl không

  kubectl-version:
    type: string
    default: "v1.28.0"
    description: Phiên bản kubectl cần cài

steps:
  # Cài kubectl nếu cần
  - when:
      condition: << parameters.install-kubectl >>
      steps:
        - run:
            name: "Cài kubectl << parameters.kubectl-version >>"
            command: |
              curl -LO "https://dl.k8s.io/release/<< parameters.kubectl-version >>/bin/linux/amd64/kubectl"
              chmod +x kubectl
              sudo mv kubectl /usr/local/bin/
              kubectl version --client

  # Thiết lập credentials theo cloud provider
  - when:
      condition:
        equal: ["aws", << parameters.cloud-provider >>]
      steps:
        - run:
            name: Thiết lập AWS credentials
            command: |
              aws sts get-caller-identity  # Kiểm tra credentials hợp lệ

  - when:
      condition:
        equal: ["gcp", << parameters.cloud-provider >>]
      steps:
        - run:
            name: Thiết lập GCP credentials
            command: |
              echo $GCP_SERVICE_KEY | \
                gcloud auth activate-service-account --key-file=-
              gcloud config set project $GCP_PROJECT_ID

  - when:
      condition:
        equal: ["azure", << parameters.cloud-provider >>]
      steps:
        - run:
            name: Thiết lập Azure credentials
            command: |
              az login --service-principal \
                -u $AZURE_CLIENT_ID \
                -p $AZURE_CLIENT_SECRET \
                --tenant $AZURE_TENANT_ID
```

---

## Viết Jobs

### Job Đầy Đủ Với Parameters

```yaml
# src/jobs/build-deploy.yml
description: >
  Job kết hợp build Docker image và deploy lên Kubernetes.
  Phù hợp cho dự án nhỏ hoặc khi không cần tách riêng hai bước.

executor: default

parameters:
  app-name:
    type: string
    description: Tên ứng dụng

  docker-registry:
    type: string
    description: URL của Docker registry (ví dụ ecr, gcr, docker hub)

  k8s-cluster:
    type: string
    description: Tên Kubernetes cluster

  k8s-namespace:
    type: string
    default: "default"

  cloud-provider:
    type: enum
    enum: ["aws", "gcp", "azure"]
    default: "aws"

  post-deploy-slack-channel:
    type: string
    default: ""
    description: Kênh Slack nhận thông báo (để trống nếu không cần)

steps:
  - checkout

  - setup:
      cloud-provider: << parameters.cloud-provider >>

  - run:
      name: Build Docker image
      command: |
        IMAGE="<< parameters.docker-registry >>/<< parameters.app-name >>:$CIRCLE_SHA1"
        docker build -t $IMAGE .
        docker push $IMAGE
        echo "export DEPLOYED_IMAGE=$IMAGE" >> $BASH_ENV

  - deploy:
      cluster-name: << parameters.k8s-cluster >>
      namespace: << parameters.k8s-namespace >>

  - when:
      condition: << parameters.post-deploy-slack-channel >>
      steps:
        - run:
            name: Thông báo Slack
            command: |
              curl -X POST $SLACK_WEBHOOK_URL \
                -H 'Content-type: application/json' \
                -d "{\"channel\": \"<< parameters.post-deploy-slack-channel >>\", \"text\": \"✅ Deploy thành công: $CIRCLE_SHA1\"}"
```

---

## Viết Executors

```yaml
# src/executors/default.yml
description: Executor mặc định — môi trường deploy với các công cụ cần thiết

parameters:
  image-tag:
    type: string
    default: "stable"
    description: Tag của base image

docker:
  - image: "cimg/base:<< parameters.image-tag >>"

resource_class: small


# src/executors/with-db.yml
description: Executor có kèm PostgreSQL service cho integration tests

parameters:
  node-version:
    type: string
    default: "20.0"
  postgres-version:
    type: string
    default: "15"

docker:
  - image: "cimg/node:<< parameters.node-version >>"
  - image: "cimg/postgres:<< parameters.postgres-version >>"
    environment:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb

resource_class: medium
```

---

## Parameters Nâng Cao

### Các Kiểu Parameter

```yaml
parameters:
  # string — Chuỗi văn bản
  app-name:
    type: string
    default: "my-app"
    description: Tên ứng dụng

  # boolean — Giá trị đúng/sai
  run-tests:
    type: boolean
    default: true

  # integer — Số nguyên
  parallelism:
    type: integer
    default: 4

  # enum — Một trong các giá trị cho phép
  environment:
    type: enum
    enum: ["dev", "staging", "production"]
    default: "staging"

  # steps — Danh sách các bước (dùng trong commands)
  pre-deploy-steps:
    type: steps
    default: []
    description: Các steps tùy chỉnh chạy trước khi deploy

  # env_var_name — Tên biến môi trường (không expose giá trị)
  aws-secret-key:
    type: env_var_name
    default: AWS_SECRET_ACCESS_KEY
    description: Tên env var chứa AWS secret key
```

### Dùng `steps` Parameter

```yaml
# src/commands/deploy-with-hooks.yml
description: Deploy với pre/post hooks tùy chỉnh

parameters:
  pre-deploy:
    type: steps
    default: []
    description: Steps chạy trước khi deploy

  post-deploy:
    type: steps
    default: []
    description: Steps chạy sau khi deploy

steps:
  # Thực thi pre-deploy steps do user cung cấp
  - steps: << parameters.pre-deploy >>

  - run:
      name: Deploy ứng dụng
      command: kubectl apply -f k8s/

  # Thực thi post-deploy steps do user cung cấp
  - steps: << parameters.post-deploy >>
```

```yaml
# Config của user — dùng steps parameter
jobs:
  deploy:
    steps:
      - my-orb/deploy-with-hooks:
          pre-deploy:
            - run: echo "Chạy database migration trước"
            - run: npm run migrate
          post-deploy:
            - run: echo "Chạy smoke tests sau khi deploy"
            - run: npm run test:smoke
```

---

## Kiểm Thử Orb

### 1. Validate Syntax — Kiểm Tra Cú Pháp

```bash
# Pack và validate
circleci orb pack src/ > /tmp/orb.yml
circleci orb validate /tmp/orb.yml
```

### 2. Integration Testing — Kiểm Thử Tích Hợp Thực Tế

Tạo pipeline riêng trong `.circleci/config.yml` của repository orb:

```yaml
# .circleci/config.yml (trong orb repository)
version: 2.1

# Import phiên bản dev của chính orb đang phát triển
orbs:
  # @dev:alpha là phiên bản dev local (publish trước khi test)
  my-company-deploy: my-company/deploy@dev:alpha
  orb-tools: circleci/orb-tools@12.0.4

workflows:
  # Workflow 1: Lint và validate
  lint-pack:
    jobs:
      - orb-tools/lint:
          filters: &all-branches
            tags:
              only: /.*/

      - orb-tools/pack:
          filters: *all-branches

      - orb-tools/publish:
          orb-name: my-company/deploy
          vcs-type: github
          pub-type: dev                  # Publish phiên bản dev trước
          filters: *all-branches
          requires:
            - orb-tools/lint
            - orb-tools/pack

  # Workflow 2: Integration test
  integration-test:
    jobs:
      # Test command setup
      - test-setup-aws:
          filters: *all-branches

      # Test command deploy
      - test-deploy-staging:
          filters: *all-branches

      # Test toàn bộ job
      - my-company-deploy/build-deploy:
          app-name: test-app
          docker-registry: $TEST_REGISTRY
          k8s-cluster: test-cluster
          filters: *all-branches

  # Workflow 3: Publish phiên bản production (chỉ khi có tag)
  publish-production:
    jobs:
      - orb-tools/publish:
          orb-name: my-company/deploy
          vcs-type: github
          pub-type: production
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/

jobs:
  test-setup-aws:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      # Kiểm thử command setup với AWS
      - my-company-deploy/setup:
          cloud-provider: aws
          install-kubectl: true
      - run:
          name: Kiểm tra kubectl được cài đúng
          command: kubectl version --client

  test-deploy-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - my-company-deploy/deploy:
          cluster-name: $TEST_CLUSTER
          namespace: test
          wait-for-completion: false     # Không cần chờ trong test
```

### 3. Orb Tools — Bộ Công Cụ Phát Triển Orb

```bash
# Cài orb-tools để hỗ trợ development
# (được dùng qua orb trong CircleCI, không cần cài local)

# Các lệnh CLI thường dùng
circleci orb create my-namespace/my-orb     # Tạo orb mới trong namespace
circleci orb pack src/ > orb.yml            # Gộp source files
circleci orb validate orb.yml               # Kiểm tra syntax
circleci orb publish orb.yml my-namespace/my-orb@dev:alpha  # Publish dev
circleci orb source my-namespace/my-orb@dev:alpha           # Xem source
```

---

## Publish Orb Lên Registry

### Bước 1: Thiết Lập Namespace

```bash
# Tạo namespace (làm một lần cho tổ chức)
# Cần quyền admin của tổ chức
circleci namespace create my-company github my-github-org
```

### Bước 2: Tạo Orb

```bash
# Tạo orb trong namespace
circleci orb create my-company/deploy
```

### Bước 3: Publish Phiên Bản Dev

```bash
# Publish để test trước khi release
circleci orb pack src/ > orb.yml
circleci orb publish orb.yml my-company/deploy@dev:alpha

# Kiểm tra
circleci orb source my-company/deploy@dev:alpha
```

### Bước 4: Publish Phiên Bản Production

```bash
# Publish phiên bản ổn định với semantic version
circleci orb publish promote my-company/deploy@dev:alpha patch
# → tạo phiên bản 0.0.1 (nếu chưa có version nào)

# Hoặc publish trực tiếp với version cụ thể
circleci orb publish orb.yml my-company/deploy@1.0.0
```

### Bước 5: Quản Lý Phiên Bản

```bash
# Xem tất cả phiên bản
circleci orb list-versions my-company/deploy

# Xem thông tin orb
circleci orb info my-company/deploy

# Xem source của phiên bản cụ thể
circleci orb source my-company/deploy@1.0.0
```

### Private Orbs — Orb Riêng Tư

Nếu không muốn public orb, có thể tạo **unlisted orb** — orb không liệt kê công khai:

```bash
# Tạo orb unlisted (vẫn cần đúng namespace/name để import)
# Trong Organization Settings → Orbs → Allow Private Orbs
circleci orb create my-company/internal-deploy --private
```

---

## Orb Development Pipeline

### Flow Hoàn Chỉnh Khi Phát Triển Orb

```
Developer viết code
        ↓
git push feature branch
        ↓
CircleCI chạy pipeline
        ↓
┌─────────────────────────────────┐
│  Workflow: lint-pack             │
│  1. orb-tools/lint               │
│     → Kiểm tra YAML syntax       │
│     → Kiểm tra best practices    │
│  2. orb-tools/pack               │
│     → Gộp src/ thành orb.yml     │
│  3. Publish dev version          │
│     → my-company/orb@dev:branch  │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  Workflow: integration-test      │
│  4. Test từng command/job        │
│  5. Test trên môi trường thực    │
└─────────────────────────────────┘
        ↓ (nếu tất cả xanh)
Code review và merge vào main
        ↓
git tag v1.2.0
        ↓
┌─────────────────────────────────┐
│  Workflow: publish-production    │
│  6. Publish v1.2.0 lên registry  │
└─────────────────────────────────┘
        ↓
Các project import my-company/orb@1.2.0
```

---

## Best Practices

### Thiết Kế Orb Tốt

```
1. Một orb chỉ làm một việc tốt (Single Responsibility)
   ✅ my-company/k8s-deploy — chỉ lo về Kubernetes deploy
   ❌ my-company/everything — làm đủ thứ

2. Đặt tên rõ ràng, nhất quán
   ✅ setup-credentials, deploy-app, verify-health
   ❌ step1, doThing, helper

3. Parameters có default values hợp lý
   ✅ namespace: default: "default"
   ❌ namespace: # Bắt buộc nhưng gần như lúc nào cũng là "default"

4. Viết description đầy đủ cho orb, commands, parameters

5. Cung cấp examples trong src/examples/

6. Đừng hardcode secrets — dùng env_var_name parameter type
   ✅ aws-key: type: env_var_name  
   ❌ aws-key: type: string  # User có thể paste key thẳng vào
```

### Versioning Strategy — Chiến Lược Quản Lý Phiên Bản

```
MAJOR (x.0.0): Breaking changes — thay đổi phá vỡ tương thích
  → Xóa parameter
  → Đổi tên command
  → Thay đổi behavior mặc định

MINOR (0.x.0): Backward-compatible new features — thêm tính năng
  → Thêm command mới
  → Thêm parameter mới có default value
  → Thêm job mới

PATCH (0.0.x): Backward-compatible bug fixes — sửa lỗi
  → Fix bug không thay đổi interface
  → Cập nhật dependency version
  → Cải thiện error messages
```

### Changelog — Nhật Ký Thay Đổi

Luôn duy trì CHANGELOG.md:

```markdown
# Changelog

## [1.2.0] - 2026-05-18
### Added
- Command `verify-health` kiểm tra deployment health sau khi deploy
- Parameter `timeout-minutes` cho command `deploy`

### Fixed
- Sửa lỗi command `setup` khi region chứa ký tự đặc biệt

## [1.1.0] - 2026-04-10
### Added
- Hỗ trợ GCP (GKE) trong addition to AWS (EKS)

## [1.0.0] - 2026-03-01
### Initial release
- Command `setup` — thiết lập credentials
- Command `deploy` — deploy lên Kubernetes
- Job `build-deploy` — combined build và deploy job
```

---

## Câu Hỏi Phỏng Vấn

### Q: Bạn sẽ thiết kế một custom orb cho tổ chức như thế nào?

**Trả lời mẫu:**
"Khi được yêu cầu chuẩn hóa quy trình deploy cho 15 microservices, tôi đã tạo internal orb `company/k8s-deploy` với 3 commands chính: `setup-cloud-credentials` (hỗ trợ AWS/GCP qua parameter), `docker-build-push` (build và push image), và `k8s-rolling-deploy` (zero-downtime deploy). Tôi dùng `env_var_name` parameter type để không ai vô tình hardcode credentials. Sau khi publish, mỗi service chỉ cần 20-30 dòng config thay vì 100+ dòng."

### Q: Sự khác biệt giữa dev version và production version của orb?

**Trả lời:**
- **Dev version** (ví dụ `@dev:alpha`): Dùng để test trong CI pipeline trước khi release. Có thể overwrite, không có số phiên bản cố định. Mặc định tự xóa sau 90 ngày.
- **Production version** (ví dụ `@1.2.0`): Immutable — không thể thay đổi hay xóa. Vĩnh viễn trên registry. Dùng trong production pipelines.

### Q: Làm thế nào để handle breaking changes trong orb mà không phá vỡ các project hiện tại?

**Trả lời:**
1. **Increment MAJOR version** — `v1.x.x` → `v2.0.0`
2. **Duy trì v1 một thời gian** — tiếp tục nhận security fixes
3. **Migration guide** — tài liệu rõ ràng những gì thay đổi
4. **Deprecation warning** — thêm thông báo deprecated trong v1 trước khi xóa
5. **Communication** — thông báo cho tất cả teams sử dụng orb

### Q: Tại sao dùng `env_var_name` parameter type thay vì `string`?

**Trả lời:**
`env_var_name` không truyền giá trị secrets vào config YAML — nó chỉ truyền **tên** của biến môi trường. Điều này ngăn người dùng vô tình paste giá trị secret thẳng vào config và commit lên git. Orb sau đó đọc giá trị từ môi trường runtime thay vì từ config.

---

## 🔗 Điều Hướng

- **Trước:** [3-inline-orbs.md](./3-inline-orbs.md) — Inline Orbs
- **Tiếp theo:** [05-optimization/](../05-optimization/) — Tối Ưu Hóa Pipeline
- **Tham khảo:** [CircleCI Orb Developer Guide](https://circleci.com/docs/orb-author-intro/)

---

**Cập Nhật Lần Cuối:** 2026-05-18 | **Phiên Bản:** 1.0
