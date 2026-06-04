# 2. Certified Orbs — Orb Được Chứng Nhận

> Certified Orbs là các orbs do CircleCI chính thức phát triển và bảo trì, đảm bảo chất lượng cao nhất và an toàn khi dùng trong môi trường production.

---

## 📚 Mục Lục

1. [aws-cli Orb](#aws-cli-orb)
2. [docker Orb](#docker-orb)
3. [node Orb](#node-orb)
4. [kubernetes Orb](#kubernetes-orb)
5. [slack Orb](#slack-orb)
6. [python Orb](#python-orb)
7. [go Orb](#go-orb)
8. [Browser Tools Orb](#browser-tools-orb)
9. [Tổng Hợp So Sánh](#tổng-hợp-so-sánh)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## aws-cli Orb

**Registry:** `circleci/aws-cli` | **Dùng khi:** Tương tác với AWS services

### Cài Đặt Và Xác Thực

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  deploy-to-aws:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout

      # Phương thức 1: OIDC — OpenID Connect — Khuyến Nghị (không cần static keys)
      - aws-cli/setup:
          role-arn: arn:aws:iam::123456789012:role/circleci-deploy-role
          region: ap-southeast-1
          session-duration: "1800"      # Thời gian hết hạn session (giây)

      # Phương thức 2: Access Key (cần thiết lập env vars)
      # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY trong Context
      - aws-cli/setup:
          region: ap-southeast-1

      # Sau khi setup, dùng AWS CLI bình thường
      - run:
          name: Deploy lên S3
          command: |
            aws s3 sync ./dist s3://my-production-bucket \
              --delete \
              --cache-control "max-age=31536000"

      - run:
          name: Invalidate CloudFront cache
          command: |
            aws cloudfront create-invalidation \
              --distribution-id $CLOUDFRONT_DISTRIBUTION_ID \
              --paths "/*"
```

### Deploy Lên ECS — Elastic Container Service

```yaml
jobs:
  deploy-ecs:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/setup:
          role-arn: $AWS_DEPLOY_ROLE_ARN
      - run:
          name: Cập nhật ECS Task Definition
          command: |
            # Lấy task definition hiện tại
            TASK_DEFINITION=$(aws ecs describe-task-definition \
              --task-definition my-app \
              --query 'taskDefinition' \
              --output json)

            # Cập nhật image mới
            NEW_TASK_DEF=$(echo $TASK_DEFINITION | jq \
              --arg IMAGE "123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/my-app:$CIRCLE_SHA1" \
              '.containerDefinitions[0].image = $IMAGE')

            # Đăng ký task definition mới
            aws ecs register-task-definition \
              --family my-app \
              --cli-input-json "$NEW_TASK_DEF"

            # Cập nhật service
            aws ecs update-service \
              --cluster production \
              --service my-app-service \
              --task-definition my-app \
              --force-new-deployment
```

### Push Lên ECR — Elastic Container Registry

```yaml
orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  push-to-ecr:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker
      - aws-cli/setup:
          role-arn: $AWS_ECR_PUSH_ROLE_ARN
      - run:
          name: Build và push image lên ECR
          command: |
            ECR_REGISTRY="123456789012.dkr.ecr.ap-southeast-1.amazonaws.com"

            # Đăng nhập vào ECR
            aws ecr get-login-password --region ap-southeast-1 | \
              docker login --username AWS --password-stdin $ECR_REGISTRY

            # Build image
            docker build -t $ECR_REGISTRY/my-app:$CIRCLE_SHA1 .
            docker tag $ECR_REGISTRY/my-app:$CIRCLE_SHA1 $ECR_REGISTRY/my-app:latest

            # Push image
            docker push $ECR_REGISTRY/my-app:$CIRCLE_SHA1
            docker push $ECR_REGISTRY/my-app:latest
```

---

## docker Orb

**Registry:** `circleci/docker` | **Dùng khi:** Build, tag, push Docker images

### Build Và Push Cơ Bản

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.3.0

jobs:
  build-image:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true   # DLC — Docker Layer Caching — tăng tốc build

      # Build image
      - docker/build:
          image: my-org/my-app         # <registry>/<image-name>
          tag: $CIRCLE_SHA1            # Tag bằng commit SHA
          dockerfile: Dockerfile       # Đường dẫn tới Dockerfile

      # Push lên Docker Hub
      - docker/push:
          image: my-org/my-app
          tag: $CIRCLE_SHA1

workflows:
  build-and-push:
    jobs:
      - build-image:
          context: docker-hub-creds    # Context chứa DOCKER_LOGIN, DOCKER_PASSWORD
```

### Multi-Tag Strategy — Chiến Lược Đa Tag

```yaml
jobs:
  build-and-tag:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true
      - docker/build:
          image: my-org/my-app
          tag: $CIRCLE_SHA1
      - run:
          name: Tag nhiều phiên bản
          command: |
            # Tag thêm với latest nếu đang build từ main
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              docker tag my-org/my-app:$CIRCLE_SHA1 my-org/my-app:latest
              docker push my-org/my-app:latest
            fi

            # Tag với version nếu là Git tag
            if [ -n "$CIRCLE_TAG" ]; then
              docker tag my-org/my-app:$CIRCLE_SHA1 my-org/my-app:$CIRCLE_TAG
              docker push my-org/my-app:$CIRCLE_TAG
            fi

            docker push my-org/my-app:$CIRCLE_SHA1
```

### Job publish Tích Hợp Của docker Orb

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.3.0

workflows:
  publish:
    jobs:
      # Pre-built job — không cần định nghĩa job riêng
      - docker/publish:
          image: my-org/my-app
          tag: $CIRCLE_SHA1
          docker-username: DOCKER_USERNAME   # Tên env var chứa username
          docker-password: DOCKER_PASSWORD   # Tên env var chứa password
          dockerfile: Dockerfile.production
          path: .                            # Build context path
          filters:
            branches:
              only: main
```

---

## node Orb

**Registry:** `circleci/node` | **Dùng khi:** Dự án Node.js / JavaScript / TypeScript

### Install Packages Với Caching Tự Động

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0

jobs:
  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout

      # install-packages tự động: detect pkg manager, cache, restore cache
      - node/install-packages:
          pkg-manager: npm             # npm | yarn | pnpm
          cache-version: v2            # Thay đổi để invalidate cache cũ

      - run: npm test

  lint:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages:
          pkg-manager: npm
      - run: npm run lint

  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/
            - node_modules/
```

### Monorepo — Cài Package Ở Nhiều Thư Mục

```yaml
jobs:
  install-monorepo:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout

      # Cài root packages
      - node/install-packages:
          app-dir: .

      # Cài packages cho frontend
      - node/install-packages:
          app-dir: ./packages/frontend
          pkg-manager: yarn
          cache-version: frontend-v1

      # Cài packages cho backend
      - node/install-packages:
          app-dir: ./packages/backend
          pkg-manager: yarn
          cache-version: backend-v1
```

### Pre-built Job — test

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0

workflows:
  test:
    jobs:
      # Dùng pre-built test job của node orb
      - node/test:
          version: "20.0"             # Node.js version
          pkg-manager: npm
          run-command: npm test       # Lệnh chạy test
          cache-version: v2
```

---

## kubernetes Orb

**Registry:** `circleci/kubernetes` | **Dùng khi:** Deploy lên Kubernetes clusters

### Cài kubectl Và Deploy

```yaml
version: 2.1

orbs:
  kubernetes: circleci/kubernetes@1.3.0
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  deploy-to-eks:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout

      # Thiết lập AWS credentials (để kết nối EKS)
      - aws-cli/setup:
          role-arn: $AWS_DEPLOY_ROLE_ARN

      # Cài kubectl — Kubernetes CLI
      - kubernetes/install-kubectl:
          kubectl-version: v1.28.0

      # Cấu hình kubeconfig — File Cấu Hình Kết Nối Cluster
      - run:
          name: Cấu hình kết nối EKS
          command: |
            aws eks update-kubeconfig \
              --name my-eks-cluster \
              --region ap-southeast-1

      # Áp dụng Kubernetes manifests — Tệp Khai Báo Kubernetes
      - kubernetes/create-or-update-resource:
          resource-file-path: k8s/deployment.yaml
          resource-name: deployment/my-app

      # Chờ rollout hoàn thành
      - kubernetes/rollout-status:
          watch-rollout-status: true
          resource-name: deployment/my-app
          namespace: production
```

### Rolling Update — Cập Nhật Cuộn Với Zero Downtime

```yaml
jobs:
  rolling-deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/setup:
          role-arn: $AWS_DEPLOY_ROLE_ARN
      - kubernetes/install-kubectl
      - run:
          name: Cấu hình kubeconfig
          command: |
            aws eks update-kubeconfig \
              --name my-eks-cluster \
              --region ap-southeast-1

      # Cập nhật image trong deployment
      - run:
          name: Cập nhật image
          command: |
            kubectl set image deployment/my-app \
              app=123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/my-app:$CIRCLE_SHA1 \
              --namespace=production

      # Theo dõi trạng thái rollout
      - kubernetes/rollout-status:
          resource-name: deployment/my-app
          namespace: production
          watch-timeout: 5m
```

### Deploy Lên GKE — Google Kubernetes Engine

```yaml
version: 2.1

orbs:
  kubernetes: circleci/kubernetes@1.3.0
  gcp-cli: circleci/gcp-cli@3.1.0

jobs:
  deploy-to-gke:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - gcp-cli/setup:
          version: 453.0.0
      - run:
          name: Xác thực với GKE
          command: |
            gcloud container clusters get-credentials my-gke-cluster \
              --zone asia-southeast1-a \
              --project my-gcp-project
      - kubernetes/install-kubectl
      - kubernetes/create-or-update-resource:
          resource-file-path: k8s/
          recursive: true
```

---

## slack Orb

**Registry:** `circleci/slack` | **Dùng khi:** Gửi thông báo Slack từ pipeline

### Thông Báo Cơ Bản

```yaml
version: 2.1

orbs:
  slack: circleci/slack@4.12.0

jobs:
  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - run: ./deploy.sh

      # Thông báo khi deploy thành công
      - slack/notify:
          event: pass                  # pass | fail | always
          template: success_tagged_deploy_1

      # Thông báo khi deploy thất bại
      - slack/notify:
          event: fail
          template: basic_fail_1
          mentions: "@oncall-team"     # Tag người/nhóm khi thất bại

workflows:
  deploy:
    jobs:
      - deploy:
          context: slack-secrets       # Context chứa SLACK_ACCESS_TOKEN, SLACK_DEFAULT_CHANNEL
```

### Custom Template — Mẫu Tùy Chỉnh

```yaml
jobs:
  notify-custom:
    docker:
      - image: cimg/base:stable
    steps:
      - run: ./run-tests.sh
      - slack/notify:
          event: always
          custom: |
            {
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "Pipeline $CIRCLE_PROJECT_REPONAME"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Branch:*\n$CIRCLE_BRANCH"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Commit:*\n$CIRCLE_SHA1"
                    }
                  ]
                }
              ]
            }
```

---

## python Orb

**Registry:** `circleci/python` | **Dùng khi:** Dự án Python

```yaml
version: 2.1

orbs:
  python: circleci/python@2.1.1

jobs:
  test:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout

      # Cài packages với cache tự động (hỗ trợ pip và poetry)
      - python/install-packages:
          pkg-manager: pip             # pip | poetry | pipenv
          args: -r requirements.txt
          cache-version: v2

      - run: pytest tests/ -v --tb=short

  lint:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - python/install-packages:
          pkg-manager: pip
          args: flake8 black isort
      - run: flake8 .
      - run: black --check .
```

---

## go Orb

**Registry:** `circleci/go` | **Dùng khi:** Dự án Go (Golang)

```yaml
version: 2.1

orbs:
  go: circleci/go@1.10.0

jobs:
  test:
    docker:
      - image: cimg/go:1.21
    steps:
      - checkout

      # Load Go module cache
      - go/load-cache
      - go/mod-download

      - run: go test ./... -v -race -coverprofile=coverage.out

      # Lưu cache cho lần chạy tiếp theo
      - go/save-cache

      - run: go tool cover -html=coverage.out -o coverage.html
      - store_artifacts:
          path: coverage.html
```

---

## Browser Tools Orb

**Registry:** `circleci/browser-tools` | **Dùng khi:** E2E tests — Kiểm Thử Đầu Cuối với Selenium/Cypress/Playwright

```yaml
version: 2.1

orbs:
  browser-tools: circleci/browser-tools@1.4.6
  node: circleci/node@5.1.0

jobs:
  e2e-test:
    docker:
      # Cần dùng image hỗ trợ browser
      - image: cimg/node:20.0-browsers
    steps:
      - checkout
      - node/install-packages

      # Cài Chrome và ChromeDriver — Trình Điều Khiển Chrome
      - browser-tools/install-chrome
      - browser-tools/install-chromedriver

      # Kiểm tra cài đặt
      - run:
          name: Kiểm tra Chrome
          command: |
            google-chrome --version
            chromedriver --version

      # Chạy E2E tests với Cypress
      - run:
          name: Chạy E2E tests
          command: npx cypress run --browser chrome --headless

      # Lưu screenshots và videos khi test thất bại
      - store_artifacts:
          path: cypress/screenshots
      - store_artifacts:
          path: cypress/videos
```

---

## Tổng Hợp So Sánh

| Orb | Namespace | Mục Đích Chính | Version Mới Nhất |
|-----|-----------|----------------|------------------|
| aws-cli | `circleci/aws-cli` | Tương tác AWS services | 4.x |
| docker | `circleci/docker` | Build/push Docker images | 2.x |
| node | `circleci/node` | Dự án Node.js | 5.x |
| kubernetes | `circleci/kubernetes` | Deploy lên K8s | 1.x |
| slack | `circleci/slack` | Thông báo Slack | 4.x |
| python | `circleci/python` | Dự án Python | 2.x |
| go | `circleci/go` | Dự án Go | 1.x |
| browser-tools | `circleci/browser-tools` | E2E browser testing | 1.x |
| gcp-cli | `circleci/gcp-cli` | Tương tác GCP | 3.x |
| terraform | `circleci/terraform` | Terraform plan/apply | 3.x |

---

## Câu Hỏi Phỏng Vấn

### Q: Bạn sử dụng orb nào thường xuyên nhất trong dự án?

**Trả lời mẫu:**
"Trong dự án microservices tại công ty, tôi dùng thường xuyên 3 orbs: `aws-cli` để thiết lập OIDC credentials và deploy lên ECS, `docker` để build và push image lên ECR, và `slack` để thông báo team khi deploy production. Kết hợp 3 orbs này giúp pipeline deployment chỉ cần khoảng 30-40 dòng YAML so với 150+ dòng nếu tự viết."

### Q: Tại sao bạn chọn OIDC trong aws-cli orb thay vì Access Keys?

**Trả lời:**
- OIDC — OpenID Connect không cần long-lived static credentials
- Tokens có thời hạn ngắn (15-60 phút), tự hết hạn
- Không cần rotate secrets định kỳ
- Audit trail rõ ràng hơn trong AWS CloudTrail
- Giảm attack surface — Bề Mặt Tấn Công khi có credential leak

### Q: node/install-packages khác gì so với tự viết `npm ci`?

**Trả lời:**
`node/install-packages` tích hợp sẵn:
1. **Cache tự động** — tự tạo cache key dựa trên `package-lock.json`, tự restore và save
2. **Detect package manager** — tự phát hiện npm/yarn/pnpm
3. **Frozen lockfile** — mặc định dùng `npm ci` thay vì `npm install`
4. **Multi-directory support** — hỗ trợ cài package ở nhiều thư mục (monorepo)

Nếu tự viết phải thêm ~20 dòng YAML cho caching.

### Q: Làm thế nào để cập nhật kubernetes deployment với zero downtime?

**Trả lời:**
Dùng kubernetes orb kết hợp với `RollingUpdate` strategy trong Kubernetes:
1. `kubectl set image` — cập nhật image mới trong deployment
2. Kubernetes tự rolling update — thay thế pods từng cái một
3. `kubernetes/rollout-status` — chờ và kiểm tra rollout thành công
4. Nếu thất bại, `kubectl rollout undo` để rollback

---

## 🔗 Điều Hướng

- **Trước:** [1-using-orbs.md](./1-using-orbs.md) — Cách dùng orbs
- **Tiếp theo:** [3-inline-orbs.md](./3-inline-orbs.md) — Inline Orbs
- **Tham khảo:** [07-integration/](../07-integration/) — Tích hợp AWS/GCP chi tiết

---

**Cập Nhật Lần Cuối:** 2026-05-18 | **Phiên Bản:** 1.0
