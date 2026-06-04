# 3. Inline Orbs — Orb Nội Tuyến

> **Inline Orb** là orb được định nghĩa thẳng bên trong file `config.yml` thay vì publish lên CircleCI Orb Registry. Không cần công cụ đặc biệt, không cần tài khoản publisher — chỉ cần viết YAML.

---

## 📚 Mục Lục

1. [Inline Orb Là Gì](#inline-orb-là-gì)
2. [Cú Pháp Khai Báo](#cú-pháp-khai-báo)
3. [Commands Trong Inline Orb](#commands-trong-inline-orb)
4. [Jobs Trong Inline Orb](#jobs-trong-inline-orb)
5. [Executors Trong Inline Orb](#executors-trong-inline-orb)
6. [Kết Hợp Inline Và External Orbs](#kết-hợp-inline-và-external-orbs)
7. [Use Cases Thực Tế](#use-cases-thực-tế)
8. [Inline Orb vs Reusable Commands](#inline-orb-vs-reusable-commands)
9. [Khi Nào Nên Dùng Inline Orb](#khi-nào-nên-dùng-inline-orb)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Inline Orb Là Gì

### Định Nghĩa

Inline Orb là một orb được viết và đóng gói hoàn toàn bên trong `config.yml`. Nó có đầy đủ cấu trúc của một orb thực sự (commands, jobs, executors, examples) nhưng **không được lưu trên registry**.

```
External Orb:   config.yml ──import──→ CircleCI Registry → orb code
Inline Orb:     config.yml ──contains──→ [orb code ở đây luôn]
```

### Mục Đích Chính

1. **Tổ chức code** — nhóm các commands liên quan vào một namespace
2. **Prototype** — thử nghiệm trước khi tách thành orb riêng
3. **Internal reuse** — dùng lại logic trong cùng một config phức tạp
4. **Đóng gói logic** — giúp phần `jobs` và `workflows` gọn hơn

---

## Cú Pháp Khai Báo

### Cấu Trúc Cơ Bản

```yaml
version: 2.1

orbs:
  # Inline orb được khai báo trong section orbs
  # Giống như external orb nhưng thay vì "namespace/name@version"
  # bạn cung cấp object chứa nội dung orb
  my-orb:
    # Khai báo commands của orb
    commands:
      my-command:
        parameters:
          param1:
            type: string
            default: "default-value"
        steps:
          - run:
              name: Thực hiện lệnh
              command: echo << parameters.param1 >>

    # Khai báo jobs của orb
    jobs:
      my-job:
        parameters:
          env:
            type: string
        docker:
          - image: cimg/base:stable
        steps:
          - my-orb/my-command:
              param1: << parameters.env >>

    # Khai báo executors của orb
    executors:
      my-executor:
        docker:
          - image: cimg/base:stable
        resource_class: medium
```

### Dùng Inline Orb Trong Config

```yaml
version: 2.1

orbs:
  utils:                            # Tên alias của inline orb
    commands:
      setup-env:
        parameters:
          env-name:
            type: string
        steps:
          - run:
              name: "Thiết lập môi trường: << parameters.env-name >>"
              command: |
                echo "ENV=<< parameters.env-name >>" >> $BASH_ENV
                source $BASH_ENV

jobs:
  build:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      # Dùng command từ inline orb — cú pháp giống external orb
      - utils/setup-env:
              env-name: "staging"
```

---

## Commands Trong Inline Orb

### Ví Dụ: Nhóm Các Deployment Commands

```yaml
version: 2.1

orbs:
  deploy:
    commands:
      # Command 1: Cài đặt công cụ deployment
      install-tools:
        parameters:
          helm-version:
            type: string
            default: "3.13.0"
        steps:
          - run:
              name: Cài Helm — Kubernetes Package Manager
              command: |
                curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | \
                  DESIRED_VERSION="v<< parameters.helm-version >>" bash
          - run:
              name: Cài kubectl
              command: |
                curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
                chmod +x kubectl
                sudo mv kubectl /usr/local/bin/

      # Command 2: Kết nối tới cluster
      connect-cluster:
        parameters:
          cluster-name:
            type: string
          region:
            type: string
            default: "ap-southeast-1"
        steps:
          - run:
              name: "Kết nối tới cluster: << parameters.cluster-name >>"
              command: |
                aws eks update-kubeconfig \
                  --name << parameters.cluster-name >> \
                  --region << parameters.region >>

      # Command 3: Deploy bằng Helm
      helm-deploy:
        parameters:
          release-name:
            type: string
          chart-path:
            type: string
            default: "./charts/app"
          namespace:
            type: string
            default: "default"
          values-file:
            type: string
            default: "values.yaml"
        steps:
          - run:
              name: "Helm deploy: << parameters.release-name >>"
              command: |
                helm upgrade --install << parameters.release-name >> \
                  << parameters.chart-path >> \
                  --namespace << parameters.namespace >> \
                  --values << parameters.values-file >> \
                  --set image.tag=$CIRCLE_SHA1 \
                  --wait \
                  --timeout 5m

      # Command 4: Kiểm tra health sau deploy
      verify-deployment:
        parameters:
          deployment-name:
            type: string
          namespace:
            type: string
            default: "default"
        steps:
          - run:
              name: "Kiểm tra deployment: << parameters.deployment-name >>"
              command: |
                kubectl rollout status deployment/<< parameters.deployment-name >> \
                  --namespace << parameters.namespace >> \
                  --timeout=5m

jobs:
  deploy-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      # Sử dụng các commands từ inline orb
      - deploy/install-tools:
          helm-version: "3.13.0"
      - deploy/connect-cluster:
          cluster-name: "staging-cluster"
      - deploy/helm-deploy:
          release-name: "my-app-staging"
          namespace: "staging"
          values-file: "helm/values-staging.yaml"
      - deploy/verify-deployment:
          deployment-name: "my-app"
          namespace: "staging"

  deploy-production:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - deploy/install-tools
      - deploy/connect-cluster:
          cluster-name: "production-cluster"
      - deploy/helm-deploy:
          release-name: "my-app-production"
          namespace: "production"
          values-file: "helm/values-production.yaml"
      - deploy/verify-deployment:
          deployment-name: "my-app"
          namespace: "production"
```

---

## Jobs Trong Inline Orb

### Ví Dụ: Nhóm Các Testing Jobs

```yaml
version: 2.1

orbs:
  testing:
    jobs:
      # Pre-built job cho unit tests
      unit-test:
        parameters:
          node-version:
            type: string
            default: "20.0"
          test-command:
            type: string
            default: "npm test"
        docker:
          - image: "cimg/node:<< parameters.node-version >>"
        steps:
          - checkout
          - restore_cache:
              keys:
                - npm-{{ checksum "package-lock.json" }}
          - run: npm ci
          - save_cache:
              key: npm-{{ checksum "package-lock.json" }}
              paths:
                - ~/.npm
          - run:
              name: Chạy unit tests
              command: << parameters.test-command >>
          - store_test_results:
              path: test-results/
          - store_artifacts:
              path: coverage/

      # Pre-built job cho integration tests
      integration-test:
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
        steps:
          - checkout
          - run: npm ci
          - run:
              name: Chờ PostgreSQL
              command: |
                for i in $(seq 1 10); do
                  nc -z localhost 5432 && break
                  sleep 2
                done
          - run:
              name: Chạy integration tests
              command: npm run test:integration
              environment:
                DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

workflows:
  full-test-suite:
    jobs:
      # Dùng pre-built jobs từ inline orb
      - testing/unit-test:
          node-version: "20.0"
          test-command: "npm run test:unit -- --ci"

      - testing/integration-test:
          requires:
            - testing/unit-test
          node-version: "20.0"
          postgres-version: "15"
```

---

## Executors Trong Inline Orb

### Ví Dụ: Chuẩn Hóa Executors Cho Toàn Project

```yaml
version: 2.1

orbs:
  envs:
    executors:
      # Executor cho Node.js development
      node-dev:
        docker:
          - image: cimg/node:20.0
        resource_class: medium        # 2 vCPU, 4GB RAM
        environment:
          NODE_ENV: development

      # Executor cho Node.js production build
      node-prod:
        docker:
          - image: cimg/node:20.0
        resource_class: large         # 4 vCPU, 8GB RAM
        environment:
          NODE_ENV: production

      # Executor cho deployment tools
      deploy-env:
        docker:
          - image: cimg/base:stable
        resource_class: small         # 1 vCPU, 2GB RAM — tiết kiệm chi phí
        environment:
          AWS_DEFAULT_REGION: ap-southeast-1

jobs:
  lint:
    executor: envs/node-dev          # Dùng executor từ inline orb
    steps:
      - checkout
      - run: npm ci
      - run: npm run lint

  test:
    executor: envs/node-dev
    steps:
      - checkout
      - run: npm ci
      - run: npm test

  build:
    executor: envs/node-prod          # Executor mạnh hơn cho build
    steps:
      - checkout
      - run: npm ci
      - run: npm run build

  deploy:
    executor: envs/deploy-env         # Executor nhỏ tiết kiệm credit
    steps:
      - checkout
      - run: ./deploy.sh
```

---

## Kết Hợp Inline Và External Orbs

Inline orbs và external orbs có thể dùng cùng nhau, thậm chí inline orbs có thể tham chiếu external orbs.

```yaml
version: 2.1

orbs:
  # External orbs
  aws-cli: circleci/aws-cli@4.0.0
  slack: circleci/slack@4.12.0

  # Inline orb — sử dụng external orbs bên trong
  my-deploy:
    commands:
      full-deploy:
        parameters:
          environment:
            type: enum
            enum: ["staging", "production"]
            default: "staging"
          slack-channel:
            type: string
            default: "#deployments"
        steps:
          # Dùng command từ external orb bên trong inline orb command
          - aws-cli/setup:
              role-arn: $AWS_DEPLOY_ROLE_ARN
          - run:
              name: "Deploy tới << parameters.environment >>"
              command: |
                ./scripts/deploy.sh << parameters.environment >> $CIRCLE_SHA1
          # Thông báo sau khi deploy
          - slack/notify:
              event: pass
              channel: << parameters.slack-channel >>
              custom: |
                {
                  "text": "✅ Deploy << parameters.environment >> thành công: $CIRCLE_SHA1"
                }

jobs:
  deploy-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - my-deploy/full-deploy:
          environment: staging
          slack-channel: "#staging-deploys"

  deploy-production:
    docker:
      - image: cimg/base:stable
    steps:
      - my-deploy/full-deploy:
          environment: production
          slack-channel: "#production-deploys"
```

---

## Use Cases Thực Tế

### Use Case 1: Multi-Cloud Deploy

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0.0

  cloud-deploy:
    commands:
      to-aws:
        parameters:
          region:
            type: string
          bucket:
            type: string
        steps:
          - aws-cli/setup:
              role-arn: $AWS_ROLE_ARN
              region: << parameters.region >>
          - run: aws s3 sync dist/ s3://<< parameters.bucket >>

      to-gcp:
        parameters:
          bucket:
            type: string
        steps:
          - run:
              name: Thiết lập gcloud
              command: |
                echo $GCP_SERVICE_KEY | gcloud auth activate-service-account --key-file=-
          - run: gsutil -m rsync -r dist/ gs://<< parameters.bucket >>

jobs:
  deploy-multi-cloud:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run: npm run build
      - cloud-deploy/to-aws:
          region: ap-southeast-1
          bucket: my-app-ap-southeast
      - cloud-deploy/to-gcp:
          bucket: my-app-gcp-backup
```

### Use Case 2: Security Scanning Pipeline

```yaml
version: 2.1

orbs:
  security:
    commands:
      scan-dependencies:
        steps:
          - run:
              name: Quét lỗ hổng dependencies — OWASP Dependency Check
              command: |
                npm audit --audit-level=high
                if [ $? -ne 0 ]; then
                  echo "Phát hiện lỗ hổng HIGH/CRITICAL trong dependencies!"
                  exit 1
                fi

      scan-secrets:
        steps:
          - run:
              name: Quét secrets bị lộ trong code — Trufflebot
              command: |
                pip install trufflehog
                trufflehog git file://. --only-verified

      scan-docker-image:
        parameters:
          image-name:
            type: string
        steps:
          - run:
              name: "Quét Docker image: << parameters.image-name >>"
              command: |
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy image << parameters.image-name >> \
                  --exit-code 1 \
                  --severity HIGH,CRITICAL

jobs:
  security-check:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - security/scan-dependencies
      - security/scan-secrets
      - security/scan-docker-image:
          image-name: "my-org/my-app:$CIRCLE_SHA1"
```

---

## Inline Orb vs Reusable Commands

Đây là câu hỏi hay gặp trong phỏng vấn: khi nào dùng **inline orb** vs **reusable commands** (commands ở cấp top-level)?

| Tiêu Chí | Inline Orb | Reusable Commands |
|----------|-----------|-------------------|
| **Cú pháp gọi** | `my-orb/command-name` | `command-name` |
| **Namespace** | Có — tránh xung đột tên | Không |
| **Nhóm logic** | Có thể nhóm commands + jobs + executors | Chỉ commands |
| **Độ phức tạp** | Cao hơn một chút | Đơn giản hơn |
| **Khi dùng** | Config lớn, nhiều nhóm logic | Config vừa, ít commands |
| **Prototype orb** | Lý tưởng | Không phù hợp |

### Ví Dụ So Sánh

```yaml
# Cách 1: Reusable Command (top-level)
commands:
  setup-environment:
    steps:
      - run: echo "setup"

jobs:
  build:
    steps:
      - setup-environment    # Gọi trực tiếp, không có prefix

---

# Cách 2: Inline Orb
orbs:
  utils:
    commands:
      setup-environment:
        steps:
          - run: echo "setup"

jobs:
  build:
    steps:
      - utils/setup-environment    # Gọi với prefix orb alias
```

**Quy tắc:** Dùng inline orb khi bạn muốn tổ chức config thành các nhóm logic riêng biệt (ví dụ `deploy/`, `testing/`, `security/`). Dùng reusable commands khi chỉ có vài commands không cần phân nhóm.

---

## Khi Nào Nên Dùng Inline Orb

### Nên Dùng ✅

```
1. Config.yml có > 300 dòng và cần tổ chức tốt hơn
2. Có nhóm commands liên quan chặt chẽ (deploy/, security/, test/)
3. Prototype logic trước khi tách thành external orb
4. Muốn chia sẻ executors và jobs trong cùng config
5. Logic quá cụ thể cho project, không phù hợp publish public
```

### Không Nên Dùng ❌

```
1. Config đơn giản < 150 dòng — dùng reusable commands
2. Logic cần dùng lại ở nhiều repositories — publish external orb
3. Nhóm chỉ có 1-2 commands — không đủ để tách orb
```

---

## Câu Hỏi Phỏng Vấn

### Q: Inline Orb khác External Orb ở điểm nào?

**Trả lời:**
- **Inline Orb:** Định nghĩa thẳng trong `config.yml`, không publish lên registry, chỉ dùng trong 1 project
- **External Orb:** Publish lên CircleCI Registry, có thể import từ bất kỳ project nào, có versioning riêng
- **Khi nào dùng inline:** Tổ chức config lớn, prototype, logic quá cụ thể cho project

### Q: Khi nào bạn nên chuyển từ Inline Orb sang External Orb?

**Trả lời:**
Khi:
1. Nhiều repository trong tổ chức cần dùng cùng logic
2. Logic đã ổn định và đủ generic để tái sử dụng
3. Cần versioning riêng để các team upgrade theo tiến độ khác nhau
4. Muốn chia sẻ với cộng đồng hoặc open source

### Q: Inline Orb có thể dùng commands từ external orb không?

**Trả lời:**
Có. Trong một step của inline orb command, bạn có thể tham chiếu commands từ external orbs đã được import. Ví dụ: trong inline orb `my-deploy`, command `full-deploy` có thể gọi `aws-cli/setup` nếu `aws-cli` orb được khai báo trong section `orbs:`.

---

## 🔗 Điều Hướng

- **Trước:** [2-certified-orbs.md](./2-certified-orbs.md) — Certified Orbs
- **Tiếp theo:** [4-custom-orb-development.md](./4-custom-orb-development.md) — Phát Triển Custom Orb

---

**Cập Nhật Lần Cuối:** 2026-05-18 | **Phiên Bản:** 1.0
