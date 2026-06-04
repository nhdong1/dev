# Monorepo Strategy — Chiến Lược CI/CD Cho Monorepo

> Monorepo — Kho Mã Nguồn Hợp Nhất là kiến trúc lưu trữ mã nguồn của nhiều project, service, hoặc package trong một repository duy nhất. Chiến lược CI/CD cho monorepo đòi hỏi kết hợp nhiều kỹ thuật nâng cao của CircleCI để đảm bảo tốc độ, hiệu quả và khả năng mở rộng.

---

## 📋 Mục Lục

1. [Monorepo Là Gì Và Thách Thức CI/CD](#1-monorepo-là-gì-và-thách-thức-cicd)
2. [So Sánh Các Chiến Lược](#2-so-sánh-các-chiến-lược)
3. [Kiến Trúc CI/CD Tham Chiếu](#3-kiến-trúc-cicd-tham-chiếu)
4. [Chiến Lược 1: Dynamic Config + Path Filtering](#4-chiến-lược-1-dynamic-config--path-filtering)
5. [Chiến Lược 2: Workflow Có Điều Kiện](#5-chiến-lược-2-workflow-có-điều-kiện)
6. [Quản Lý Phụ Thuộc Giữa Các Service](#6-quản-lý-phụ-thuộc-giữa-các-service)
7. [Deploy Strategy — Chiến Lược Triển Khai](#7-deploy-strategy--chiến-lược-triển-khai)
8. [Caching Trong Monorepo](#8-caching-trong-monorepo)
9. [Monorepo Ở Quy Mô Lớn](#9-monorepo-ở-quy-mô-lớn)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Monorepo Là Gì Và Thách Thức CI/CD

### Định Nghĩa Và Ví Dụ Điển Hình

```
Polyrepo (nhiều repo):          Monorepo (một repo):
  github.com/co/frontend          github.com/co/platform/
  github.com/co/api-service         ├── apps/
  github.com/co/auth-service        │   ├── frontend/
  github.com/co/shared-lib          │   ├── mobile/
                                    │   └── admin/
  (4 repo riêng biệt)              ├── services/
                                    │   ├── api/
                                    │   ├── auth/
                                    │   └── payment/
                                    └── libs/
                                        ├── shared-ui/
                                        └── utils/
                                    (1 repo duy nhất)
```

**Công ty dùng monorepo nổi tiếng:** Google (Bazel), Meta (Buck), Twitter, Airbnb, Lyft, Shopify.

### Lợi Ích Của Monorepo

```
✅ Atomic commits — Commit nguyên tử: thay đổi nhiều service trong 1 commit
✅ Dependency management — Quản lý phụ thuộc: shared lib luôn đồng bộ
✅ Code reuse — Tái sử dụng code: dễ dàng dùng chung utils
✅ Refactoring — Cấu trúc lại: đổi API một chỗ, compiler báo lỗi toàn bộ
✅ Visibility — Khả năng nhìn thấy: developer hiểu toàn bộ hệ thống
```

### Thách Thức CI/CD Của Monorepo

```
❌ Vấn đề 1: Build toàn bộ khi chỉ 1 service thay đổi
   → Giải pháp: Path Filtering + Dynamic Config

❌ Vấn đề 2: Test suite khổng lồ (10.000+ test)
   → Giải pháp: Test Splitting + Parallelism

❌ Vấn đề 3: Dependency graph phức tạp
   → Giải pháp: Dependency aware build tools (Nx, Turborepo, Bazel)

❌ Vấn đề 4: Deploy coordination — phối hợp deploy
   → Giải pháp: Workflow orchestration + manual approval gates

❌ Vấn đề 5: Cache invalidation — vô hiệu hóa cache
   → Giải pháp: Fine-grained cache keys per service
```

---

## 2. So Sánh Các Chiến Lược

| Chiến Lược | Độ Phức Tạp | Hiệu Quả | Phù Hợp Khi |
| ---------- | ----------- | --------- | ------------ |
| **Luôn build tất cả** | Thấp | Rất thấp | Monorepo nhỏ (<5 service), build nhanh |
| **Workflow có điều kiện** | Trung bình | Trung bình | 5-15 service, logic đơn giản |
| **Dynamic Config + Path Filtering** | Cao | Cao | 15+ service, build lâu |
| **Nx / Turborepo affected** | Cao | Rất cao | Đã dùng build tools chuyên dụng |
| **Bazel** | Rất cao | Xuất sắc | Google-scale, ngàn service |

---

## 3. Kiến Trúc CI/CD Tham Chiếu

Kiến trúc được khuyến nghị cho monorepo trung bình (10-50 service):

```
┌─────────────────────────────────────────────────────────────────┐
│  Tầng 1: DETECTION — Phát Hiện Thay Đổi                        │
│                                                                 │
│  setup-workflow (Dynamic Config)                                │
│    ├── Phân tích git diff                                       │
│    ├── Xác định service bị ảnh hưởng (kể cả transitive deps)   │
│    └── Set pipeline parameters                                  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  Tầng 2: BUILD & TEST — Xây Dựng Và Kiểm Thử                  │
│                                                                 │
│  Chạy song song chỉ service bị ảnh hưởng:                      │
│    ├── lint + unit test (parallelism: 4)                       │
│    ├── integration test (parallelism: 2)                       │
│    └── build artifact (Docker image / binary)                  │
└───────────────────────────────┬─────────────────────────────────┘
                                │ requires: all-tests-pass
┌───────────────────────────────▼─────────────────────────────────┐
│  Tầng 3: STAGING DEPLOY — Triển Khai Staging                   │
│                                                                 │
│  Deploy song song lên staging environment:                      │
│    ├── deploy-service-a → staging                               │
│    └── deploy-service-b → staging                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │ requires: staging-smoke-tests
┌───────────────────────────────▼─────────────────────────────────┐
│  Tầng 4: MANUAL APPROVAL — Duyệt Thủ Công                     │
│                                                                 │
│  approve-production:                                            │
│    type: approval             ← Engineer review và approve      │
└───────────────────────────────┬─────────────────────────────────┘
                                │ requires: approval (main only)
┌───────────────────────────────▼─────────────────────────────────┐
│  Tầng 5: PRODUCTION DEPLOY — Triển Khai Production             │
│                                                                 │
│  Blue/Green hoặc Canary deploy                                  │
│    └── Rollback tự động nếu health check thất bại              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Chiến Lược 1: Dynamic Config + Path Filtering

Đây là chiến lược được khuyến nghị cho monorepo quy mô trung bình trở lên.

### Cấu Trúc File

```
.circleci/
├── config.yml              ← Setup config (entry point)
├── continue-config.yml     ← Main config với conditional jobs
└── scripts/
    └── generate_config.py  ← Tùy chọn: script sinh config phức tạp
```

### Setup Config

```yaml
# .circleci/config.yml
version: 2.1
setup: true

orbs:
  path-filtering: circleci/path-filtering@1.0.0

workflows:
  # Workflow này chỉ làm một việc: phát hiện thay đổi
  detect-changes:
    jobs:
      - path-filtering/filter:
          name: detect-service-changes
          base-revision: main
          config-path: .circleci/continue-config.yml
          mapping: |
            # Apps
            apps/frontend/.*           build-frontend     true
            apps/mobile/.*             build-mobile       true
            apps/admin/.*              build-admin        true

            # Services
            services/api/.*            build-api          true
            services/auth/.*           build-auth         true
            services/payment/.*        build-payment      true
            services/notification/.*   build-notification true

            # Shared libs — khi thay đổi, các service dùng lib đều cần rebuild
            libs/shared-ui/.*          build-frontend     true
            libs/shared-ui/.*          build-admin        true
            libs/shared-ui/.*          build-mobile       true
            libs/utils/.*              build-api          true
            libs/utils/.*              build-auth         true
            libs/utils/.*              build-payment      true

            # Infrastructure
            infrastructure/.*          plan-infra         true

            # CI config change → rebuild everything
            \.circleci/.*              build-all          true
```

### Continue Config

```yaml
# .circleci/continue-config.yml
version: 2.1

parameters:
  build-frontend:
    type: boolean
    default: false
  build-mobile:
    type: boolean
    default: false
  build-admin:
    type: boolean
    default: false
  build-api:
    type: boolean
    default: false
  build-auth:
    type: boolean
    default: false
  build-payment:
    type: boolean
    default: false
  build-notification:
    type: boolean
    default: false
  plan-infra:
    type: boolean
    default: false
  build-all:
    type: boolean
    default: false

orbs:
  node: circleci/node@5.1.0
  aws-cli: circleci/aws-cli@4.0.0

# ─── EXECUTORS ───────────────────────────────────────────────

executors:
  node-executor:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

  python-executor:
    docker:
      - image: cimg/python:3.12
    resource_class: medium

  go-executor:
    docker:
      - image: cimg/go:1.22
    resource_class: medium

# ─── COMMANDS — LỆNH TÁI SỬ DỤNG ──────────────────────────

commands:
  setup-aws:
    steps:
      - aws-cli/setup:
          role_arn: "${AWS_ROLE_ARN}"

  docker-build-push:
    parameters:
      service-name:
        type: string
      service-dir:
        type: string
    steps:
      - run:
          name: Build Docker image cho << parameters.service-name >>
          command: |
            IMAGE_TAG="${CIRCLE_SHA1:0:7}"
            docker build \
              -t "${ECR_REGISTRY}/<< parameters.service-name >>:${IMAGE_TAG}" \
              -t "${ECR_REGISTRY}/<< parameters.service-name >>:latest" \
              << parameters.service-dir >>
      - run:
          name: Push image lên ECR — Elastic Container Registry
          command: |
            IMAGE_TAG="${CIRCLE_SHA1:0:7}"
            aws ecr get-login-password | docker login --username AWS --password-stdin "${ECR_REGISTRY}"
            docker push "${ECR_REGISTRY}/<< parameters.service-name >>:${IMAGE_TAG}"
            docker push "${ECR_REGISTRY}/<< parameters.service-name >>:latest"

# ─── JOBS ───────────────────────────────────────────────────

jobs:
  # ── Frontend ──────────────────────────────────────────────
  build-test-frontend:
    executor: node-executor
    steps:
      - checkout
      - restore_cache:
          keys:
            - frontend-deps-v1-{{ checksum "apps/frontend/package-lock.json" }}
      - node/install-packages:
          app-dir: apps/frontend
      - save_cache:
          key: frontend-deps-v1-{{ checksum "apps/frontend/package-lock.json" }}
          paths:
            - apps/frontend/node_modules
      - run:
          working_directory: apps/frontend
          name: Lint + Type check
          command: npm run lint && npm run type-check
      - run:
          working_directory: apps/frontend
          name: Unit test
          command: npm test -- --coverage
      - run:
          working_directory: apps/frontend
          name: Build production bundle
          command: npm run build
      - persist_to_workspace:
          root: apps/frontend
          paths:
            - dist/

  deploy-frontend-staging:
    executor: node-executor
    steps:
      - attach_workspace:
          at: /tmp/frontend
      - setup-aws
      - run:
          name: Deploy frontend lên S3 staging
          command: |
            aws s3 sync /tmp/frontend/dist/ \
              s3://myapp-staging-frontend \
              --delete --cache-control "max-age=0,no-cache"

  # ── API Service ───────────────────────────────────────────
  build-test-api:
    executor: go-executor
    parallelism: 2
    steps:
      - checkout
      - restore_cache:
          keys:
            - api-go-modules-v1-{{ checksum "services/api/go.sum" }}
      - run:
          working_directory: services/api
          name: Download modules
          command: go mod download
      - save_cache:
          key: api-go-modules-v1-{{ checksum "services/api/go.sum" }}
          paths:
            - /root/go/pkg/mod
      - run:
          working_directory: services/api
          name: Build và test API
          command: |
            go vet ./...
            go test -v -race ./... 2>&1 | tee /tmp/test-output.txt
            go build ./...

  # ── Payment Service (yêu cầu đặc biệt: PCI-DSS) ──────────
  build-test-payment:
    machine: true
    resource_class: mycompany/secure-runner  # Self-hosted runner trong secure network
    steps:
      - checkout
      - run:
          working_directory: services/payment
          name: Kiểm thử payment service trong mạng nội bộ
          command: |
            pip install -r requirements.txt
            pytest tests/ -v --cov=payment

  # ── Infrastructure ────────────────────────────────────────
  terraform-plan:
    docker:
      - image: hashicorp/terraform:1.7
    steps:
      - checkout
      - setup-aws
      - run:
          working_directory: infrastructure
          name: Terraform init và plan
          command: |
            terraform init
            terraform plan -out=tfplan -detailed-exitcode
      - store_artifacts:
          path: infrastructure/tfplan
          destination: terraform-plan

  terraform-apply:
    docker:
      - image: hashicorp/terraform:1.7
    steps:
      - checkout
      - setup-aws
      - run:
          working_directory: infrastructure
          command: |
            terraform init
            terraform apply -auto-approve

# ─── WORKFLOWS ─────────────────────────────────────────────

workflows:
  monorepo-ci:
    jobs:
      # ── Frontend jobs ─────────────────────────────────────
      - build-test-frontend:
          context: aws-staging
          when:
            or:
              - << pipeline.parameters.build-frontend >>
              - << pipeline.parameters.build-all >>

      - deploy-frontend-staging:
          context: aws-staging
          requires:
            - build-test-frontend
          filters:
            branches:
              only: [main, develop]
          when:
            or:
              - << pipeline.parameters.build-frontend >>
              - << pipeline.parameters.build-all >>

      # ── API jobs ──────────────────────────────────────────
      - build-test-api:
          when:
            or:
              - << pipeline.parameters.build-api >>
              - << pipeline.parameters.build-all >>

      # ── Payment jobs ──────────────────────────────────────
      - build-test-payment:
          context: payment-secrets
          when:
            or:
              - << pipeline.parameters.build-payment >>
              - << pipeline.parameters.build-all >>

      # ── Infrastructure jobs ───────────────────────────────
      - terraform-plan:
          context: aws-prod
          when:
            or:
              - << pipeline.parameters.plan-infra >>
              - << pipeline.parameters.build-all >>

      - approve-terraform:
          type: approval
          requires:
            - terraform-plan
          filters:
            branches:
              only: main
          when:
            or:
              - << pipeline.parameters.plan-infra >>
              - << pipeline.parameters.build-all >>

      - terraform-apply:
          context: aws-prod
          requires:
            - approve-terraform
          filters:
            branches:
              only: main
```

---

## 5. Chiến Lược 2: Workflow Có Điều Kiện

Đơn giản hơn Dynamic Config, phù hợp với monorepo nhỏ hơn (5-15 service):

```yaml
# .circleci/config.yml (không cần setup: true)
version: 2.1

parameters:
  # Có thể trigger thủ công qua API với parameters
  force-build-api:
    type: boolean
    default: false

jobs:
  detect-changes:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Phát hiện service nào thay đổi và set parameters
          command: |
            CHANGED=$(git diff --name-only origin/main...HEAD 2>/dev/null || git diff --name-only HEAD~1)

            # Ghi kết quả vào file để các job sau dùng
            echo "API_CHANGED=$(echo "$CHANGED" | grep -c "^services/api/" || true)" > /tmp/changes
            echo "AUTH_CHANGED=$(echo "$CHANGED" | grep -c "^services/auth/" || true)" >> /tmp/changes
            cat /tmp/changes
      - persist_to_workspace:
          root: /tmp
          paths:
            - changes

  build-api:
    docker:
      - image: cimg/go:1.22
    steps:
      - attach_workspace:
          at: /tmp
      - run:
          name: Kiểm tra có cần build không
          command: |
            source /tmp/changes
            if [ "$API_CHANGED" = "0" ] && [ "<< pipeline.parameters.force-build-api >>" = "false" ]; then
              echo "API không thay đổi, bỏ qua build"
              circleci-agent step halt  # Halt job sớm, không fail
            fi
      - checkout
      - run: cd services/api && go test ./... && go build ./...

workflows:
  monorepo-workflow:
    jobs:
      - detect-changes
      - build-api:
          requires: [detect-changes]
```

---

## 6. Quản Lý Phụ Thuộc Giữa Các Service

### Dependency Graph — Đồ Thị Phụ Thuộc

```python
# scripts/dependency_resolver.py
"""
Tự động tính toán danh sách service cần rebuild dựa trên dependency graph.
"""

# Định nghĩa dependency graph
# key: service name, value: danh sách service/lib mà nó phụ thuộc vào
DEPENDENCY_GRAPH = {
    "api-gateway":   ["shared-proto", "auth-client-lib"],
    "auth-service":  ["shared-proto", "shared-db-lib"],
    "payment-service": ["shared-proto", "auth-client-lib", "shared-db-lib"],
    "notification-service": ["shared-proto", "email-template-lib"],
    "frontend": ["shared-ui", "shared-types"],
    "admin": ["shared-ui", "shared-types"],
}

# Ngược lại: ai phụ thuộc vào lib này?
def build_reverse_graph(graph):
    reverse = {}
    for service, deps in graph.items():
        for dep in deps:
            if dep not in reverse:
                reverse[dep] = []
            reverse[dep].append(service)
    return reverse

REVERSE_GRAPH = build_reverse_graph(DEPENDENCY_GRAPH)

def get_all_affected(changed_paths):
    """
    Cho danh sách file thay đổi, trả về tất cả service/lib cần rebuild.
    Bao gồm cả transitive dependencies — phụ thuộc bắc cầu.
    """
    # Tìm service/lib trực tiếp bị ảnh hưởng
    directly_affected = set()
    for path in changed_paths:
        for component in list(DEPENDENCY_GRAPH.keys()) + list(REVERSE_GRAPH.keys()):
            if path.startswith(f"services/{component}/") or \
               path.startswith(f"libs/{component}/") or \
               path.startswith(f"apps/{component}/"):
                directly_affected.add(component)

    # Lan truyền: ai phụ thuộc vào các component đã ảnh hưởng?
    all_affected = set(directly_affected)
    queue = list(directly_affected)

    while queue:
        component = queue.pop(0)
        dependents = REVERSE_GRAPH.get(component, [])
        for dependent in dependents:
            if dependent not in all_affected:
                all_affected.add(dependent)
                queue.append(dependent)

    return all_affected


if __name__ == "__main__":
    import sys
    import json

    # Đọc danh sách file thay đổi từ stdin
    changed_files = sys.stdin.read().strip().split("\n")
    affected = get_all_affected(changed_files)

    # Xuất dưới dạng JSON cho CircleCI parameters
    params = {service: True for service in affected}
    print(json.dumps(params, indent=2))
```

### Tích Hợp Với Nx — Build Tool Cho Monorepo

```yaml
# Dùng Nx để phát hiện affected projects
jobs:
  nx-affected:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages
      - run:
          name: Build và test chỉ các project bị ảnh hưởng
          command: |
            # Nx tự động tính dependency graph và chạy chỉ affected
            npx nx affected:build --base=origin/main --head=HEAD
            npx nx affected:test --base=origin/main --head=HEAD
      - run:
          name: Lint affected
          command: npx nx affected:lint --base=origin/main --head=HEAD
```

---

## 7. Deploy Strategy — Chiến Lược Triển Khai

### Environment Promotion — Thăng Cấp Môi Trường

```
Feature Branch → PR → Staging → Canary → Production

              review-app       full staging   10% traffic   100% traffic
              (per-PR)         (develop)      (main)        (main)
```

```yaml
workflows:
  monorepo-deploy:
    jobs:
      - build-and-test:
          # Chạy trên tất cả branch
          filters:
            branches:
              only: /.*/

      - deploy-review-app:
          # Deploy review app cho mỗi PR
          requires: [build-and-test]
          filters:
            branches:
              ignore: [main, develop, /release\/.*/]

      - deploy-staging:
          # Deploy lên staging khi merge vào develop
          requires: [build-and-test]
          filters:
            branches:
              only: [develop]

      - run-e2e-tests:
          # End-to-end test — Kiểm thử đầu cuối trên staging
          requires: [deploy-staging]

      - approve-canary:
          # Duyệt deploy canary — thử nghiệm với 10% traffic
          type: approval
          requires: [run-e2e-tests]
          filters:
            branches:
              only: main

      - deploy-canary:
          context: aws-prod
          requires: [approve-canary]
          # Canary deployment: 10% traffic
          filters:
            branches:
              only: main

      - canary-health-check:
          requires: [deploy-canary]

      - approve-production:
          type: approval
          requires: [canary-health-check]

      - deploy-production:
          context: aws-prod
          requires: [approve-production]
```

### Service Versioning — Quản Lý Phiên Bản Service

```bash
# scripts/tag-and-version.sh
# Tạo version tag cho service khi deploy

SERVICE=$1
COMMIT=${CIRCLE_SHA1:0:7}
BRANCH=${CIRCLE_BRANCH}
TIMESTAMP=$(date +%Y%m%d%H%M%S)

if [ "$BRANCH" = "main" ]; then
    # Production: dùng semantic version từ CHANGELOG
    VERSION=$(cat services/$SERVICE/VERSION)
    TAG="v${VERSION}"
else
    # Non-production: dùng branch + commit
    TAG="${BRANCH//\//-}-${COMMIT}"
fi

echo "Tagging $SERVICE với version: $TAG"
docker tag "${ECR_REGISTRY}/${SERVICE}:${COMMIT}" "${ECR_REGISTRY}/${SERVICE}:${TAG}"
docker push "${ECR_REGISTRY}/${SERVICE}:${TAG}"
```

---

## 8. Caching Trong Monorepo

### Cache Strategy Per Service — Chiến Lược Cache Riêng Cho Từng Service

```yaml
commands:
  restore-service-cache:
    parameters:
      service:
        type: string
      cache-version:
        type: string
        default: "v1"
    steps:
      - restore_cache:
          # Cache key theo service, tránh conflict giữa các service
          keys:
            - << parameters.cache-version >>-<< parameters.service >>-{{ checksum "services/<< parameters.service >>/package-lock.json" }}
            - << parameters.cache-version >>-<< parameters.service >>-

  save-service-cache:
    parameters:
      service:
        type: string
      cache-version:
        type: string
        default: "v1"
    steps:
      - save_cache:
          key: << parameters.cache-version >>-<< parameters.service >>-{{ checksum "services/<< parameters.service >>/package-lock.json" }}
          paths:
            - services/<< parameters.service >>/node_modules

jobs:
  build-service-a:
    steps:
      - checkout
      - restore-service-cache:
          service: service-a
      - run: cd services/service-a && npm ci
      - save-service-cache:
          service: service-a
      - run: cd services/service-a && npm test
```

### Global Cache Cho Shared Dependencies — Cache Dùng Chung

```yaml
jobs:
  install-shared-deps:
    steps:
      - checkout
      - restore_cache:
          keys:
            - shared-deps-v1-{{ checksum "libs/shared-ui/package-lock.json" }}-{{ checksum "libs/utils/package-lock.json" }}
      - run:
          name: Cài đặt shared lib dependencies
          command: |
            cd libs/shared-ui && npm ci
            cd libs/utils && npm ci
      - save_cache:
          key: shared-deps-v1-{{ checksum "libs/shared-ui/package-lock.json" }}-{{ checksum "libs/utils/package-lock.json" }}
          paths:
            - libs/shared-ui/node_modules
            - libs/utils/node_modules
      - persist_to_workspace:
          root: libs
          paths:
            - shared-ui/node_modules
            - utils/node_modules

  build-frontend:
    steps:
      - checkout
      - attach_workspace:
          at: libs   # Lấy shared deps đã cài sẵn
      - run: cd apps/frontend && npm ci && npm run build
```

---

## 9. Monorepo Ở Quy Mô Lớn

### Vấn Đề Khi Monorepo Phát Triển

```
50+ services → Dynamic Config config.yml rất dài
100+ jobs → Pipeline visualization khó đọc
1000+ test → Test splitting không đủ
10 team → Ownership và access control phức tạp
```

### Giải Pháp: Config Được Sinh Từ Template

```python
# scripts/generate_monorepo_config.py
"""
Sinh .circleci/continue-config.yml tự động từ danh sách service.
Chạy script này mỗi khi thêm service mới thay vì sửa config thủ công.
"""
import os
import yaml

SERVICES = [
    {"name": "api-gateway", "lang": "go", "runner": "cloud"},
    {"name": "auth-service", "lang": "python", "runner": "cloud"},
    {"name": "payment-service", "lang": "python", "runner": "secure-runner"},
    # Thêm service mới ở đây
]

APPS = [
    {"name": "frontend", "lang": "node"},
    {"name": "admin", "lang": "node"},
]

def generate_config():
    config = {
        "version": "2.1",
        "parameters": {},
        "jobs": {},
        "workflows": {
            "monorepo-ci": {"jobs": []}
        }
    }

    # Sinh parameters và jobs cho mỗi service
    for service in SERVICES + APPS:
        name = service["name"]
        param_name = f"build-{name}"

        # Pipeline parameter
        config["parameters"][param_name] = {
            "type": "boolean",
            "default": False
        }

        # Build job
        config["jobs"][f"build-{name}"] = build_job_template(service)

        # Conditional job trong workflow
        config["workflows"]["monorepo-ci"]["jobs"].append({
            f"build-{name}": {
                "when": {
                    "or": [
                        f"<< pipeline.parameters.{param_name} >>",
                        "<< pipeline.parameters.build-all >>"
                    ]
                }
            }
        })

    # build-all parameter
    config["parameters"]["build-all"] = {"type": "boolean", "default": False}

    return yaml.dump(config, default_flow_style=False, allow_unicode=True)


def build_job_template(service):
    """Tạo job definition dựa trên ngôn ngữ và runner type."""
    lang_images = {
        "go": "cimg/go:1.22",
        "python": "cimg/python:3.12",
        "node": "cimg/node:20.0",
    }

    job = {
        "steps": [
            "checkout",
            {"run": {"name": f"Build và test {service['name']}", "command": get_build_command(service['lang'], service['name'])}}
        ]
    }

    if service.get("runner") == "secure-runner":
        job["machine"] = True
        job["resource_class"] = f"mycompany/secure-runner"
    else:
        job["docker"] = [{"image": lang_images[service["lang"]]}]

    return job


def get_build_command(lang, service_name):
    commands = {
        "go": f"cd services/{service_name} && go test ./... && go build ./...",
        "python": f"cd services/{service_name} && pip install -r requirements.txt && pytest",
        "node": f"cd apps/{service_name} && npm ci && npm test && npm run build",
    }
    return commands[lang]


if __name__ == "__main__":
    print(generate_config())
```

### Access Control — Kiểm Soát Truy Cập Theo Team

```yaml
# Dùng Restricted Contexts để giới hạn ai được deploy service nào
# Mỗi team chỉ có quyền với Context của họ

workflows:
  deploy:
    jobs:
      - deploy-payment:
          context:
            - payment-team-secrets     # Chỉ payment team member có quyền
            - aws-prod-payment         # AWS credentials riêng cho payment
          filters:
            branches:
              only: main

      - deploy-auth:
          context:
            - security-team-secrets   # Chỉ security team có quyền
            - aws-prod-auth
          filters:
            branches:
              only: main
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Monorepo là gì và những thách thức CI/CD chính là gì?**

> Monorepo là kiến trúc lưu tất cả service và lib của tổ chức trong một repository duy nhất. Thách thức CI/CD chính: (1) Thời gian build tăng theo số service — giải quyết bằng Path Filtering; (2) Test suite khổng lồ — giải quyết bằng Parallelism và Test Splitting; (3) Dependency graph phức tạp — cần công cụ như Nx/Turborepo; (4) Deploy coordination giữa service — cần workflow orchestration; (5) Cache management phức tạp hơn.

**Q: Làm thế nào để tránh build tất cả service khi chỉ 1 service thay đổi?**

> Dùng Dynamic Config kết hợp Path Filtering orb. Setup workflow phân tích git diff, ánh xạ đường dẫn file thay đổi vào pipeline parameters, sau đó continuation config dùng `when` conditions để chỉ chạy job của service bị ảnh hưởng. Quan trọng là cũng phải tính transitive dependencies — ví dụ khi shared lib thay đổi, tất cả service dùng lib đó cũng phải rebuild.

**Q: Làm thế nào để xử lý khi shared library thay đổi ảnh hưởng nhiều service?**

> Trong Path Filtering mapping, ánh xạ cùng một path pattern của shared library tới nhiều pipeline parameters. Hoặc dùng một flag `build-all: true` và tất cả job có điều kiện `or: [<specific-flag>, build-all]`. Với quy mô lớn hơn, có thể viết script dependency resolver để tự động tính transitive dependencies và sinh mapping tương ứng.

**Q: Monorepo vs Polyrepo — khi nào nên chọn cái nào?**

> Monorepo phù hợp khi: các service có nhiều shared code, cần atomic commit, team nhỏ đến trung bình, ưu tiên code reuse và refactoring toàn hệ thống. Polyrepo phù hợp khi: service hoàn toàn độc lập, nhiều team với tech stack rất khác nhau, cần giới hạn access control theo repo level, hoặc service có vòng đời release độc lập. Thực tế, nhiều công ty chọn "monorepo light" — một repo per domain, không phải một repo duy nhất cho toàn công ty.

**Q: Bạn sẽ thiết kế CI/CD pipeline cho monorepo 20 service như thế nào?**

> Tôi sẽ thiết kế theo 5 tầng: (1) Detection — dùng Dynamic Config + Path Filtering để xác định service bị ảnh hưởng kể cả transitive deps; (2) Build & Test — chạy song song chỉ service cần rebuild, dùng Parallelism cho test suite lớn; (3) Artifact Publishing — push Docker image lên ECR với tag bao gồm git SHA; (4) Staging Deploy — deploy song song lên staging, chạy integration test tự động; (5) Production Deploy — Manual approval gate, Canary deployment với tự động rollback nếu health check thất bại. Tất cả secrets quản lý qua Contexts theo team ownership.

---

## 🔗 Xem Thêm

- [1-dynamic-config.md](1-dynamic-config.md) — Chi tiết Dynamic Config
- [2-path-filtering.md](2-path-filtering.md) — Chi tiết Path Filtering
- [3-matrix-jobs.md](3-matrix-jobs.md) — Matrix Jobs cho cross-platform testing
- [4-self-hosted-runner.md](4-self-hosted-runner.md) — Runner tự quản lý cho yêu cầu đặc biệt
- [05-optimization/](../05-optimization/) — Caching và parallelism chi tiết

---

**Cập Nhật Lần Cuối:** 2026-05-20
