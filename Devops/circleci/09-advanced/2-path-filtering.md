# Path Filtering — Lọc Theo Đường Dẫn

> Path Filtering — Lọc Theo Đường Dẫn là kỹ thuật trong CircleCI Dynamic Config cho phép pipeline quyết định job nào sẽ chạy dựa trên các file được thay đổi trong commit. Đây là giải pháp thiết yếu cho monorepo — kho mã nguồn hợp nhất khi cần tránh build toàn bộ project mỗi khi chỉ có một phần nhỏ thay đổi.

---

## 📋 Mục Lục

1. [Tại Sao Cần Path Filtering?](#1-tại-sao-cần-path-filtering)
2. [Path-Filtering Orb — Gói Lọc Đường Dẫn](#2-path-filtering-orb--gói-lọc-đường-dẫn)
3. [Cấu Hình Cơ Bản](#3-cấu-hình-cơ-bản)
4. [Mapping Rules — Quy Tắc Ánh Xạ](#4-mapping-rules--quy-tắc-ánh-xạ)
5. [Continue Config — Config Tiếp Nối](#5-continue-config--config-tiếp-nối)
6. [Ví Dụ Thực Tế: Monorepo Full-Stack](#6-ví-dụ-thực-tế-monorepo-full-stack)
7. [Path Filtering Thủ Công Với Git Diff](#7-path-filtering-thủ-công-với-git-diff)
8. [Chiến Lược Phụ Thuộc Giữa Các Service](#8-chiến-lược-phụ-thuộc-giữa-các-service)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Path Filtering?

### Vấn Đề Với Monorepo Không Có Path Filtering

```
Monorepo với 10 service:
  services/
    api/          → build: 3 phút
    auth/         → build: 4 phút
    payment/      → build: 5 phút
    notification/ → build: 2 phút
    analytics/    → build: 6 phút
    ...

Không có Path Filtering:
  Developer sửa 1 dòng trong services/api/
  → Toàn bộ 10 service được build
  → Tổng thời gian: 40+ phút
  → Credit tiêu thụ: 10x cần thiết
  → Developer chờ quá lâu, productivity — năng suất giảm
```

### Lợi Ích Sau Khi Áp Dụng Path Filtering

```
Với Path Filtering:
  Developer sửa 1 dòng trong services/api/
  → Chỉ services/api/ được build: 3 phút
  → Thời gian giảm: 40 phút → 3 phút (giảm 92.5%)
  → Credit tiêu thụ: 1/10 so với trước
  → Developer nhận feedback nhanh hơn
```

---

## 2. Path-Filtering Orb — Gói Lọc Đường Dẫn

Orb `circleci/path-filtering` là giải pháp chính thức của CircleCI, kết hợp Dynamic Config để tự động:

1. So sánh file thay đổi với git diff
2. Ánh xạ — mapping path pattern sang pipeline parameters
3. Truyền parameters vào continuation config

### Cài Đặt Yêu Cầu

1. Bật Dynamic Config trên Project Settings
2. Khai báo `setup: true` trong config.yml
3. Import path-filtering orb

```yaml
# .circleci/config.yml
version: 2.1
setup: true

orbs:
  path-filtering: circleci/path-filtering@1.0.0
```

---

## 3. Cấu Hình Cơ Bản

### Setup Config — File Khởi Tạo

```yaml
# .circleci/config.yml
version: 2.1
setup: true

orbs:
  path-filtering: circleci/path-filtering@1.0.0

workflows:
  setup:
    jobs:
      - path-filtering/filter:
          # base-revision: nhánh / commit để so sánh
          base-revision: main

          # mapping: ánh xạ regex đường dẫn → pipeline parameter
          # Cú pháp mỗi dòng: <regex-pattern> <parameter-name> <value>
          mapping: |
            services/api/.*    build-api    true
            services/auth/.*   build-auth   true
            services/worker/.* build-worker true
            libs/shared/.*     build-all    true

          # config-path: file config cho giai đoạn 2
          config-path: .circleci/continue-config.yml
```

### Giải Thích Mapping Syntax — Cú Pháp Ánh Xạ

```
services/api/.*   build-api   true
│                 │           │
│                 │           └── Giá trị gán cho parameter
│                 └── Tên pipeline parameter trong continue-config
└── Regex pattern khớp với đường dẫn file thay đổi
```

**Quy tắc:** Nếu BẤT KỲ file nào thay đổi khớp với regex, parameter đó được set thành giá trị đã khai báo.

---

## 4. Mapping Rules — Quy Tắc Ánh Xạ

### Các Pattern Regex Thường Dùng

```yaml
mapping: |
  # Khớp mọi file trong thư mục services/api/
  services/api/.*              build-api         true

  # Khớp cụ thể file package.json
  services/api/package\.json   rebuild-api-deps  true

  # Khớp file trong thư mục con bất kỳ
  libs/.*                      build-libs        true

  # Khớp file extension cụ thể
  .*\.proto                    build-proto       true

  # Khớp file ở root
  Dockerfile                   build-base-image  true
  docker-compose\.yml          update-compose    true

  # Khớp thư mục config
  \.circleci/.*                rebuild-all       true

  # Khớp nhiều pattern với OR logic
  # (mỗi dòng là OR riêng biệt)
  services/payment/.*          build-payment     true
  services/billing/.*          build-payment     true
```

### Xử Lý Shared Libraries — Thư Viện Dùng Chung

```yaml
mapping: |
  # Khi shared lib thay đổi, cần rebuild tất cả service dùng nó
  libs/shared-utils/.*    build-api          true
  libs/shared-utils/.*    build-auth         true
  libs/shared-utils/.*    build-notification true
  libs/shared-utils/.*    build-worker       true

  # Hoặc dùng một flag duy nhất "build-all"
  libs/.*                 build-all          true
```

---

## 5. Continue Config — Config Tiếp Nối

File này nhận các parameters từ path-filtering và dùng chúng để điều kiện hóa job nào chạy.

```yaml
# .circleci/continue-config.yml
version: 2.1

# Khai báo tất cả parameters có thể nhận
parameters:
  build-api:
    type: boolean
    default: false
  build-auth:
    type: boolean
    default: false
  build-worker:
    type: boolean
    default: false
  build-all:
    type: boolean
    default: false

jobs:
  build-and-test-api:
    docker:
      - image: cimg/node:20.0
    resource_class: medium
    steps:
      - checkout
      - restore_cache:
          keys:
            - api-deps-v1-{{ checksum "services/api/package-lock.json" }}
      - run:
          working_directory: services/api
          command: npm ci
      - save_cache:
          key: api-deps-v1-{{ checksum "services/api/package-lock.json" }}
          paths:
            - services/api/node_modules
      - run:
          name: Kiểm thử API service
          working_directory: services/api
          command: npm test

  build-and-test-auth:
    docker:
      - image: cimg/go:1.22
    steps:
      - checkout
      - run:
          name: Kiểm thử Auth service
          working_directory: services/auth
          command: go test ./...

  build-and-test-worker:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout
      - run:
          name: Kiểm thử Worker service
          working_directory: services/worker
          command: pip install -r requirements.txt && pytest

workflows:
  conditional-ci:
    jobs:
      # Job chỉ chạy khi build-api=true HOẶC build-all=true
      - build-and-test-api:
          when:
            or:
              - << pipeline.parameters.build-api >>
              - << pipeline.parameters.build-all >>

      - build-and-test-auth:
          when:
            or:
              - << pipeline.parameters.build-auth >>
              - << pipeline.parameters.build-all >>

      - build-and-test-worker:
          when:
            or:
              - << pipeline.parameters.build-worker >>
              - << pipeline.parameters.build-all >>
```

---

## 6. Ví Dụ Thực Tế: Monorepo Full-Stack

### Cấu Trúc Thư Mục

```
my-monorepo/
├── .circleci/
│   ├── config.yml           ← setup config (path filtering)
│   └── continue-config.yml  ← main config với điều kiện
├── frontend/
│   ├── react-app/
│   └── mobile-app/
├── backend/
│   ├── api-gateway/
│   ├── user-service/
│   └── order-service/
├── infrastructure/
│   └── terraform/
└── libs/
    ├── shared-types/
    └── ui-components/
```

### Setup Config Đầy Đủ

```yaml
# .circleci/config.yml
version: 2.1
setup: true

orbs:
  path-filtering: circleci/path-filtering@1.0.0

workflows:
  setup:
    jobs:
      - path-filtering/filter:
          base-revision: main
          config-path: .circleci/continue-config.yml
          mapping: |
            frontend/react-app/.*         build-react       true
            frontend/mobile-app/.*        build-mobile      true
            backend/api-gateway/.*        build-gateway     true
            backend/user-service/.*       build-user        true
            backend/order-service/.*      build-order       true
            infrastructure/terraform/.*   plan-terraform    true
            libs/shared-types/.*          build-react       true
            libs/shared-types/.*          build-gateway     true
            libs/shared-types/.*          build-user        true
            libs/shared-types/.*          build-order       true
            libs/ui-components/.*         build-react       true
            libs/ui-components/.*         build-mobile      true
            \.circleci/.*                 build-all         true
```

### Continue Config Đầy Đủ

```yaml
# .circleci/continue-config.yml
version: 2.1

parameters:
  build-react:
    type: boolean
    default: false
  build-mobile:
    type: boolean
    default: false
  build-gateway:
    type: boolean
    default: false
  build-user:
    type: boolean
    default: false
  build-order:
    type: boolean
    default: false
  plan-terraform:
    type: boolean
    default: false
  build-all:
    type: boolean
    default: false

orbs:
  node: circleci/node@5.1.0
  aws-cli: circleci/aws-cli@4.0.0

executors:
  node-executor:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

jobs:
  build-react-app:
    executor: node-executor
    steps:
      - checkout
      - node/install-packages:
          app-dir: frontend/react-app
      - run:
          working_directory: frontend/react-app
          command: npm run build && npm test

  build-mobile-app:
    docker:
      - image: cimg/node:20.0
    resource_class: large
    steps:
      - checkout
      - node/install-packages:
          app-dir: frontend/mobile-app
      - run:
          working_directory: frontend/mobile-app
          command: npx expo export

  build-api-gateway:
    docker:
      - image: cimg/go:1.22
    steps:
      - checkout
      - run:
          working_directory: backend/api-gateway
          command: go build ./... && go test ./...

  build-user-service:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout
      - run:
          working_directory: backend/user-service
          command: pip install -r requirements.txt && pytest -v

  build-order-service:
    docker:
      - image: cimg/java:17.0
    steps:
      - checkout
      - run:
          working_directory: backend/order-service
          command: ./gradlew test

  terraform-plan:
    docker:
      - image: hashicorp/terraform:1.7
    steps:
      - checkout
      - aws-cli/setup
      - run:
          working_directory: infrastructure/terraform
          command: |
            terraform init
            terraform plan -out=tfplan
      - store_artifacts:
          path: infrastructure/terraform/tfplan

workflows:
  conditional-build:
    jobs:
      - build-react-app:
          when:
            or:
              - << pipeline.parameters.build-react >>
              - << pipeline.parameters.build-all >>

      - build-mobile-app:
          when:
            or:
              - << pipeline.parameters.build-mobile >>
              - << pipeline.parameters.build-all >>

      - build-api-gateway:
          when:
            or:
              - << pipeline.parameters.build-gateway >>
              - << pipeline.parameters.build-all >>

      - build-user-service:
          when:
            or:
              - << pipeline.parameters.build-user >>
              - << pipeline.parameters.build-all >>

      - build-order-service:
          when:
            or:
              - << pipeline.parameters.build-order >>
              - << pipeline.parameters.build-all >>

      - terraform-plan:
          when:
            or:
              - << pipeline.parameters.plan-terraform >>
              - << pipeline.parameters.build-all >>
```

---

## 7. Path Filtering Thủ Công Với Git Diff

Khi cần logic phức tạp hơn orb hỗ trợ, có thể tự viết script phân tích git diff:

```bash
#!/bin/bash
# scripts/detect-changes.sh

# Lấy danh sách file thay đổi so với base branch
BASE_BRANCH="${CIRCLE_BASE_REVISION:-main}"
CHANGED_FILES=$(git diff --name-only "origin/${BASE_BRANCH}...HEAD" 2>/dev/null || git diff --name-only HEAD~1)

echo "=== File thay đổi ==="
echo "$CHANGED_FILES"
echo "====================="

# Khởi tạo các flag
BUILD_API=false
BUILD_AUTH=false
BUILD_ALL=false

# Phân tích từng file
while IFS= read -r file; do
  if [[ "$file" == services/api/* ]]; then
    BUILD_API=true
  fi
  if [[ "$file" == services/auth/* ]]; then
    BUILD_AUTH=true
  fi
  # Thay đổi shared lib → rebuild tất cả
  if [[ "$file" == libs/* ]]; then
    BUILD_ALL=true
  fi
  # Thay đổi CI config → rebuild tất cả
  if [[ "$file" == .circleci/* ]]; then
    BUILD_ALL=true
  fi
done <<< "$CHANGED_FILES"

# Xuất parameters dưới dạng JSON cho continuation API
cat > /tmp/pipeline-params.json << EOF
{
  "build-api": ${BUILD_API},
  "build-auth": ${BUILD_AUTH},
  "build-all": ${BUILD_ALL}
}
EOF

echo "=== Pipeline Parameters ==="
cat /tmp/pipeline-params.json
```

```yaml
# Dùng trong setup job
jobs:
  detect-and-continue:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Phát hiện thay đổi
          command: bash scripts/detect-changes.sh
      - run:
          name: Gọi continuation với parameters
          command: |
            PARAMS=$(cat /tmp/pipeline-params.json)
            curl --request POST \
              --url https://circleci.com/api/v2/pipeline/continue \
              --header "Circle-Token: ${CIRCLE_TOKEN}" \
              --header "Content-Type: application/json" \
              --data "{
                \"continuation-key\": \"${CIRCLE_CONTINUATION_KEY}\",
                \"configuration\": $(cat .circleci/continue-config.yml | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))'),
                \"parameters\": ${PARAMS}
              }"
```

---

## 8. Chiến Lược Phụ Thuộc Giữa Các Service

Một thách thức quan trọng: khi service A phụ thuộc vào service B, thay đổi B cần trigger build cả A.

### Dependency Map — Bản Đồ Phụ Thuộc

```yaml
# .circleci/config.yml - setup với dependency awareness
mapping: |
  # Khi auth-service thay đổi, api-gateway (dùng auth) cũng cần rebuild
  services/auth-service/.*     build-auth     true
  services/auth-service/.*     build-gateway  true   ← dependency

  # Khi shared proto thay đổi, tất cả service dùng proto đều rebuild
  protos/.*                    build-auth     true
  protos/.*                    build-gateway  true
  protos/.*                    build-order    true
```

### Dependency Matrix Approach — Cách Tiếp Cận Ma Trận Phụ Thuộc

```python
# scripts/dependency_matrix.py
# Tự động tính dependency graph và sinh mapping

DEPENDENCIES = {
    "api-gateway": ["auth-service", "shared-proto"],
    "order-service": ["auth-service", "payment-service", "shared-proto"],
    "notification-service": ["order-service"],
}

def get_affected_services(changed_paths):
    """Tìm tất cả service cần rebuild khi paths thay đổi."""
    directly_changed = set()
    for path in changed_paths:
        for service in DEPENDENCIES:
            if path.startswith(f"services/{service}/"):
                directly_changed.add(service)

    # Lan truyền phụ thuộc (transitive dependencies)
    affected = set(directly_changed)
    changed = True
    while changed:
        changed = False
        for service, deps in DEPENDENCIES.items():
            if any(dep in affected for dep in deps) and service not in affected:
                affected.add(service)
                changed = True

    return affected
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Path Filtering hoạt động như thế nào trong CircleCI?**

> Path Filtering dùng Dynamic Config để so sánh danh sách file thay đổi (từ git diff) với các regex pattern được định nghĩa trong mapping. Mỗi pattern được gán một pipeline parameter. Nếu pattern khớp, parameter đó được set thành giá trị đã khai báo. Những parameters này được truyền vào continuation pipeline, nơi `when` conditions dùng chúng để quyết định job nào thực sự chạy.

**Q: Khi shared library thay đổi, làm thế nào để đảm bảo tất cả service phụ thuộc đều được rebuild?**

> Trong mapping, ánh xạ cùng một đường dẫn shared library tới nhiều parameter khác nhau. Ví dụ: `libs/shared/.*  build-api  true` và `libs/shared/.*  build-auth  true`. Nếu cần logic phụ thuộc phức tạp hơn, có thể dùng một flag `build-all: true` và trong continue-config, tất cả job đều có điều kiện `or: [<job-flag>, build-all]`.

**Q: Path Filtering có giải quyết được trường hợp CI config (.circleci/) thay đổi không?**

> Có. Thêm dòng `\.circleci/.*  build-all  true` vào mapping để khi ai thay đổi CI config, toàn bộ pipeline chạy để validate thay đổi. Đây là best practice vì CI config change thường ảnh hưởng tất cả service.

**Q: Sự khác biệt giữa Path Filtering và chỉ dùng branch filter trong workflow?**

> Branch filter quyết định workflow nào chạy dựa trên tên nhánh (main, develop, feature/*). Path Filtering quyết định JOB nào chạy bên trong workflow dựa trên NỘI DUNG thay đổi (file nào được sửa). Chúng giải quyết hai bài toán khác nhau và thường được dùng kết hợp.

---

## 🔗 Xem Thêm

- [1-dynamic-config.md](1-dynamic-config.md) — Nền tảng Dynamic Config
- [5-monorepo-strategy.md](5-monorepo-strategy.md) — Chiến lược monorepo tổng thể
- [circleci/path-filtering orb](https://circleci.com/developer/orbs/orb/circleci/path-filtering)

---

**Cập Nhật Lần Cuối:** 2026-05-20
