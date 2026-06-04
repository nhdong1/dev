# Contexts — Ngữ Cảnh: Quản Lý Secrets Tập Trung

> Contexts — Ngữ Cảnh là cơ chế quản lý environment variables — biến môi trường ở cấp tổ chức (organization level), cho phép chia sẻ secrets an toàn giữa nhiều project mà vẫn kiểm soát được quyền truy cập.

## 📚 Mục Lục

1. [Contexts là gì?](#contexts-là-gì)
2. [Context vs Project Variables](#context-vs-project-variables)
3. [Tạo và Quản Lý Context](#tạo-và-quản-lý-context)
4. [Dùng Context trong Config](#dùng-context-trong-config)
5. [Context Security Groups](#context-security-groups)
6. [Restricted Contexts](#restricted-contexts)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Contexts là gì?

**Context** là một tập hợp environment variables — biến môi trường được quản lý ở cấp **tổ chức (organization)**, không phải cấp project. Điều này cho phép:

```
Không có Contexts (trước đây):
  Project A → có biến AWS_KEY riêng
  Project B → có biến AWS_KEY riêng (copy/paste)
  Project C → có biến AWS_KEY riêng (copy/paste)
  → Khi rotate key: phải cập nhật ở 3 nơi!

Có Contexts (hiện tại):
  Context "aws-deploy" chứa AWS_KEY
    ↓ inject vào
  Project A, B, C cùng dùng một context
  → Khi rotate key: cập nhật 1 lần, áp dụng mọi nơi ✅
```

### Khi Nào Dùng Context?

| Tình Huống | Giải Pháp |
|------------|-----------|
| Secrets dùng ở nhiều project | Context — chia sẻ tập trung |
| Secrets theo môi trường (staging/prod) | Context riêng cho từng môi trường |
| Giới hạn ai được dùng secrets nhạy cảm | Context Security Groups |
| Compliance — yêu cầu audit trail | Context + Audit Log |

---

## Context vs Project Variables

| Tiêu Chí | Project Variables | Context Variables |
|----------|------------------|--------------------|
| **Phạm vi** | 1 project | Toàn tổ chức |
| **Quản lý tập trung** | Không | Có |
| **Chia sẻ giữa projects** | Không thể | Dễ dàng |
| **Security Groups** | Không có | Có (Enterprise) |
| **Audit trail** | Hạn chế | Đầy đủ |
| **Khi rotate secrets** | Từng project | 1 lần |
| **Phù hợp cho** | Config không nhạy cảm | Secrets production |

### Thứ Tự Ưu Tiên — Override Order

```
Nếu cùng tên biến tồn tại ở nhiều nơi:

Built-in CircleCI vars (CIRCLE_SHA1, CIRCLE_BRANCH...)
  ↑ Không thể override
Context variables
  ↑ Override project variables
Project variables
  ↑ Override job-level vars (run: environment:)
Job-level environment (khai báo trong config.yml)
```

> ⚠️ **Lưu ý:** Context variables **override** project variables nếu trùng tên. Điều này có thể gây ra hành vi không mong muốn — hãy đặt tên biến nhất quán.

---

## Tạo và Quản Lý Context

### Qua CircleCI Dashboard — Giao Diện Web

```
1. Đăng nhập circleci.com
2. Chọn Organization (tổ chức)
3. Organization Settings → Contexts
4. Create Context → đặt tên (ví dụ: "production-aws")
5. Add Environment Variable → thêm từng secret
6. Save
```

### Qua CircleCI CLI — Giao Diện Dòng Lệnh

```bash
# Cài CircleCI CLI
curl -fLSs https://raw.githubusercontent.com/CircleCI-Public/circleci-cli/main/install.sh | bash

# Đăng nhập
circleci setup

# Danh sách các contexts
circleci context list github <org-name>

# Tạo context mới
circleci context create github <org-name> production-aws

# Xem variables trong context
circleci context show github <org-name> production-aws

# Thêm biến vào context (nhập giá trị qua stdin)
echo -n "AKIA..." | circleci context store-secret github <org-name> production-aws AWS_ACCESS_KEY_ID

# Xóa biến khỏi context
circleci context remove-secret github <org-name> production-aws AWS_ACCESS_KEY_ID
```

### Qua API — Giao Diện Lập Trình

```bash
# Lấy danh sách contexts qua REST API
curl --request GET \
  --url "https://circleci.com/api/v2/context?owner-id=<org-id>&owner-type=organization" \
  --header "Circle-Token: $CIRCLECI_TOKEN"

# Tạo context
curl --request POST \
  --url "https://circleci.com/api/v2/context" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"name": "production-aws", "owner": {"id": "<org-id>", "type": "organization"}}'
```

---

## Dùng Context trong Config

### Cú Pháp Cơ Bản

```yaml
workflows:
  deploy:
    jobs:
      - deploy-job:
          context: production-aws    # Tên context đã tạo
```

### Dùng Nhiều Contexts Cùng Lúc

```yaml
workflows:
  deploy:
    jobs:
      - deploy-job:
          context:
            - aws-credentials       # Context 1: AWS keys
            - slack-notifications   # Context 2: Slack webhook
            - datadog-monitoring    # Context 3: Datadog API key
```

> **Lưu ý:** Khi dùng nhiều contexts, nếu có biến trùng tên, context được liệt kê **sau** sẽ override context trước. Hãy đặt tên biến khác nhau giữa các contexts để tránh nhầm lẫn.

### Kết Hợp Context với Branch Filters — Bộ Lọc Nhánh

```yaml
workflows:
  ci-cd-pipeline:
    jobs:
      # Job không cần secrets
      - run-tests:
          filters:
            branches:
              only: /.*/

      # Deploy staging — chỉ từ nhánh develop
      - deploy-staging:
          context: staging-secrets
          requires: [run-tests]
          filters:
            branches:
              only: develop

      # Approval gate — Cổng Duyệt Thủ Công trước production
      - approve-production:
          type: approval
          requires: [deploy-staging]
          filters:
            branches:
              only: main

      # Deploy production — chỉ nhánh main, sau khi được duyệt
      - deploy-production:
          context: production-secrets    # Context mạnh nhất
          requires: [approve-production]
          filters:
            branches:
              only: main
```

### Truy Cập Biến Trong Job

Sau khi inject context, các biến được dùng như environment variables thông thường:

```yaml
jobs:
  deploy-production:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Cấu hình AWS CLI
          command: |
            # Biến từ context "production-secrets" tự động có sẵn
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set region $AWS_DEFAULT_REGION

      - run:
          name: Deploy lên ECS
          command: |
            aws ecs update-service \
              --cluster production \
              --service my-app \
              --force-new-deployment
```

---

## Context Security Groups

> **Tính năng này yêu cầu CircleCI Scale Plan (trả phí).**

Security Groups — Nhóm Bảo Mật cho phép **giới hạn** context chỉ có thể dùng bởi thành viên trong một GitHub/Bitbucket team nhất định.

### Cách Hoạt Động

```
Không có Security Group:
  Mọi developer trong org → đều có thể dùng context "production-secrets"

Có Security Group:
  Context "production-secrets" → giới hạn cho team "senior-devops"
    → Chỉ thành viên team "senior-devops" mới kích hoạt được job dùng context này
    → Developer bình thường cố chạy → job bị từ chối (UNAUTHORIZED)
```

### Thiết Lập Security Groups

```
Dashboard → Organization Settings → Contexts
  → Chọn context "production-secrets"
  → Security → Add Security Group
  → Chọn GitHub Team (phải connect GitHub Teams)
  → Save
```

### Ví Dụ Thực Tế — Real-World Example

```yaml
# Tình huống: Chỉ team "release-managers" mới deploy được production

workflows:
  release:
    jobs:
      - build:
          filters:
            branches:
              only: main

      - deploy-prod:
          context: production-deploy    # Context bị bảo vệ bởi Security Group
          requires: [build]
          # Nếu người trigger job không thuộc team "release-managers"
          # → CircleCI sẽ từ chối chạy job này
```

---

## Restricted Contexts

**Restricted Context** — Ngữ Cảnh Bị Hạn Chế là context chỉ được kích hoạt khi pipeline đáp ứng một số điều kiện nhất định.

### Hạn Chế Theo Nhánh (Branch-Based Restriction)

```yaml
# Cách phổ biến nhất: kết hợp context với branch filter
workflows:
  deploy:
    jobs:
      - deploy:
          context: prod-secrets
          filters:
            branches:
              only:
                - main
                - release/*    # Chỉ nhánh main hoặc release/x.x
```

### Bảo Vệ Fork PRs — Pull Request Từ Fork

Fork PR — Pull Request từ repository được fork (bản sao) là **rủi ro bảo mật**:

```
Kẻ tấn công:
  1. Fork repo của bạn
  2. Thêm code độc hại in ra $AWS_SECRET_KEY trong pipeline
  3. Tạo PR vào repo gốc
  4. Nếu pipeline chạy với context → secrets bị lộ trong logs!
```

**CircleCI mặc định không inject context vào fork PRs.** Tuy nhiên cần kiểm tra:

```yaml
# KHÔNG LÀM THẾ NÀY với secrets nhạy cảm:
workflows:
  test:
    jobs:
      - test:
          context: my-secrets     # Sẽ bị inject ngay cả với fork PRs!

# THAY VÀO ĐÓ, chỉ dùng context khi cần thiết:
workflows:
  test:
    jobs:
      - test                      # Không context → an toàn với fork PRs
      - deploy:
          context: my-secrets
          requires: [test]
          filters:
            branches:
              only: main          # Fork PRs không bao giờ merge vào main trực tiếp
```

---

## Best Practices

### Đặt Tên Context Có Ý Nghĩa

```
# Không tốt:
context: secrets
context: prod
context: env1

# Tốt:
context: aws-production-deploy
context: staging-database-creds
context: slack-alerting-webhooks
context: datadog-metrics-api
```

### Nguyên Tắc Least Privilege — Quyền Tối Thiểu

```
# Nguyên tắc: Mỗi context chỉ chứa secrets cần thiết cho 1 nhiệm vụ cụ thể

# KHÔNG TỐT: 1 context chứa tất cả
context: all-secrets
  AWS_KEY, GCP_KEY, DB_PASSWORD, SLACK_TOKEN, GITHUB_TOKEN, ...

# TỐT: Phân tách theo mục đích
context: aws-ecr-push          # Chỉ push Docker image
  AWS_ACCESS_KEY_ID
  AWS_SECRET_ACCESS_KEY
  AWS_DEFAULT_REGION

context: aws-ecs-deploy        # Chỉ deploy lên ECS
  AWS_ACCESS_KEY_ID            # Key khác, quyền khác
  AWS_SECRET_ACCESS_KEY
  ECS_CLUSTER_NAME

context: slack-notifications   # Chỉ gửi thông báo
  SLACK_WEBHOOK_URL
```

### Chiến Lược Đặt Tên Context Theo Môi Trường

```
Môi trường Dev — Development:
  dev-aws
  dev-database

Môi trường Staging — Kiểm Thử Trước Production:
  staging-aws
  staging-database
  staging-external-apis

Môi trường Production — Sản Xuất:
  prod-aws              ← Được bảo vệ bởi Security Group
  prod-database         ← Được bảo vệ bởi Security Group
  prod-external-apis    ← Được bảo vệ bởi Security Group
```

### Secret Rotation — Xoay Vòng Bí Mật

```bash
# Quy trình rotate secrets trong context:
# 1. Tạo credentials mới trong AWS/GCP
# 2. Cập nhật trong CircleCI Context (không xóa cái cũ ngay)
# 3. Chạy test pipeline để xác nhận credentials mới hoạt động
# 4. Revoke — Thu hồi credentials cũ trong AWS/GCP
# → Zero downtime rotation!

# Tự động rotate qua CircleCI API:
NEW_SECRET=$(aws iam create-access-key --user-name ci-user | jq -r '.AccessKey.SecretAccessKey')
echo -n "$NEW_SECRET" | circleci context store-secret github myorg prod-aws AWS_SECRET_ACCESS_KEY
```

### Không Hardcode Secrets Trong Config

```yaml
# ❌ SAI — Secret hiển thị trong git history và logs
jobs:
  deploy:
    steps:
      - run:
          command: |
            aws configure set aws_access_key_id AKIAIOSFODNN7EXAMPLE
            aws configure set aws_secret_access_key wJalrXUtnFEMI/K7MDENG

# ✅ ĐÚNG — Dùng biến từ context
jobs:
  deploy:
    steps:
      - run:
          command: |
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
```

---

## Ví Dụ Thực Tế — End-to-End Example

### Cấu Trúc Context cho Startup E-commerce

```
Organization: my-startup

Contexts:
  ├── aws-ecr-staging           # Push image lên staging registry
  │   ├── AWS_ACCOUNT_ID=123456789
  │   ├── AWS_ROLE_ARN_STAGING=arn:aws:iam::123:role/ci-staging
  │   └── AWS_DEFAULT_REGION=ap-southeast-1
  │
  ├── aws-ecr-production        # Push image lên production registry
  │   ├── AWS_ACCOUNT_ID=987654321
  │   ├── AWS_ROLE_ARN_PROD=arn:aws:iam::987:role/ci-prod
  │   └── AWS_DEFAULT_REGION=ap-southeast-1
  │
  ├── kubernetes-staging        # Deploy lên K8s staging cluster
  │   ├── KUBE_SERVER=https://k8s-staging.example.com
  │   └── KUBE_TOKEN=eyJhbGciOiJSUzI1NiJ9...
  │
  ├── kubernetes-production     # Deploy lên K8s production (Security Group)
  │   ├── KUBE_SERVER=https://k8s-prod.example.com
  │   └── KUBE_TOKEN=eyJhbGciOiJSUzI1NiJ9...
  │
  └── notifications             # Slack, PagerDuty
      ├── SLACK_WEBHOOK=https://hooks.slack.com/...
      └── PAGERDUTY_TOKEN=abc123
```

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.1.0

jobs:
  build-and-push:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: docker build -t $IMAGE_NAME:$CIRCLE_SHA1 .
      - aws-cli/setup
      - run:
          name: Push lên ECR
          command: |
            aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL
            docker push $IMAGE_NAME:$CIRCLE_SHA1

  deploy-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Deploy lên Kubernetes Staging
          command: |
            kubectl --server=$KUBE_SERVER --token=$KUBE_TOKEN \
              set image deployment/my-app \
              my-app=$IMAGE_NAME:$CIRCLE_SHA1

  notify-success:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Thông báo Slack
          command: |
            curl -X POST $SLACK_WEBHOOK \
              -d '{"text": "Deploy thành công! :white_check_mark:"}'

workflows:
  ci-cd:
    jobs:
      # Build & push staging image
      - build-and-push:
          context:
            - aws-ecr-staging    # AWS credentials
          name: build-staging
          filters:
            branches:
              only: develop

      # Deploy lên staging
      - deploy-staging:
          context:
            - kubernetes-staging
            - notifications
          requires: [build-staging]

      # Build & push production image
      - build-and-push:
          context:
            - aws-ecr-production
          name: build-production
          filters:
            branches:
              only: main

      # Approval gate — Cổng Duyệt Thủ Công
      - approve-production-deploy:
          type: approval
          requires: [build-production]

      # Deploy production — được bảo vệ bởi Security Group
      - deploy-production:
          context:
            - kubernetes-production   # Security Group: chỉ team "release-managers"
            - notifications
          requires: [approve-production-deploy]
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Context và Project Environment Variables khác nhau thế nào?**

> **A:** Project variables chỉ tồn tại trong một project, trong khi Context được quản lý ở cấp organization và có thể chia sẻ giữa nhiều project. Context có thêm tính năng Security Groups để giới hạn quyền truy cập, và khi cần rotate secrets, chỉ cần cập nhật ở một nơi thay vì từng project. Context variables cũng override project variables nếu trùng tên.

**Q: Khi nào nên dùng Context thay vì Project Variables?**

> **A:** Nên dùng Context khi: (1) secrets cần dùng ở nhiều project, (2) cần phân quyền ai được dùng secrets, (3) cần audit trail đầy đủ, (4) môi trường production cần bảo vệ chặt chẽ hơn. Dùng Project Variables cho config không nhạy cảm chỉ dùng trong một project.

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để rotate secrets trong Context mà không gây downtime pipeline?**

> **A:** Quy trình zero-downtime rotation: (1) Tạo credentials mới trong cloud provider (AWS/GCP), (2) Thêm credentials mới vào Context (giữ lại cái cũ), (3) Chạy test pipeline để xác nhận, (4) Sau khi xác nhận ok, revoke credentials cũ. Nếu dùng API, có thể tự động hóa bước 2 bằng `circleci context store-secret`.

**Q: Làm sao bảo vệ secrets không bị lộ qua fork PRs?**

> **A:** CircleCI mặc định không inject Context vào fork PRs. Để chắc chắn an toàn: chỉ attach Context vào các jobs thực sự cần secrets (không phải job test), và luôn kết hợp với branch filter `only: main` cho production Context. Không bao giờ attach production Context vào job test chạy trên mọi branch.

**Q: Security Groups hoạt động như thế nào?**

> **A:** Security Groups liên kết một Context với một GitHub/Bitbucket Team. Khi job cố dùng Context đó, CircleCI kiểm tra xem người trigger pipeline có thuộc team được phép hay không. Nếu không → job bị từ chối với lỗi UNAUTHORIZED. Điều này đảm bảo chỉ senior engineers/release managers mới có thể kích hoạt production deployment, ngay cả khi họ vô tình push lên branch main.

---

## 🔗 Điều Hướng

| ← Trước | Vị Trí | Tiếp → |
|---------|--------|--------|
| [06-security/README.md](./README.md) | **1-contexts.md** | [2-oidc-integration.md](./2-oidc-integration.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Độ Khó:** ⭐⭐ Trung Bình
**Thời Gian Đọc:** ~45 phút
