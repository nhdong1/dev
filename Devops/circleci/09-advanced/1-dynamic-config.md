# Dynamic Config — Cấu Hình Động

> Dynamic Config — Cấu Hình Động cho phép CircleCI tạo ra file cấu hình pipeline một cách linh hoạt tại thời điểm chạy thay vì dùng file `.circleci/config.yml` tĩnh cố định. Đây là tính năng nền tảng để xây dựng CI/CD thông minh cho monorepo và pipeline có điều kiện phức tạp.

---

## 📋 Mục Lục

1. [Dynamic Config Là Gì?](#1-dynamic-config-là-gì)
2. [Kiến Trúc Hai Giai Đoạn](#2-kiến-trúc-hai-giai-đoạn)
3. [Bật Dynamic Config](#3-bật-dynamic-config)
4. [Setup Workflow — Luồng Khởi Tạo](#4-setup-workflow--luồng-khởi-tạo)
5. [Continuation — Tiếp Nối Pipeline](#5-continuation--tiếp-nối-pipeline)
6. [Ví Dụ Thực Tế](#6-ví-dụ-thực-tế)
7. [Pipeline Parameters Trong Dynamic Config](#7-pipeline-parameters-trong-dynamic-config)
8. [Giới Hạn Và Lưu Ý](#8-giới-hạn-và-lưu-ý)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Dynamic Config Là Gì?

**Dynamic Config — Cấu Hình Động** là cơ chế cho phép pipeline CircleCI giai đoạn 1 (setup) tạo ra cấu hình YAML tùy ý rồi truyền cho giai đoạn 2 thực thi. Cấu hình được sinh ra tại runtime — thời điểm chạy, không phải lúc commit code.

### Trường Hợp Sử Dụng Điển Hình

| Bài Toán | Giải Pháp Với Dynamic Config |
| -------- | --------------------------- |
| Monorepo 20 service, mỗi commit chỉ thay đổi 1-2 service | Chỉ tạo job cho service bị ảnh hưởng |
| Pipeline logic khác nhau giữa feature branch và release branch | Sinh config khác nhau theo loại branch |
| Cần đọc file manifest để quyết định job nào cần chạy | Đọc manifest trong setup rồi sinh config |
| Số lượng service tăng động, không muốn sửa config thủ công | Sinh job list từ danh sách service đọc ở runtime |

### So Sánh Static vs Dynamic Config

```
Static Config (config.yml tĩnh):
  commit → CircleCI đọc .circleci/config.yml → chạy pipeline cố định

Dynamic Config:
  commit → CircleCI chạy setup workflow ngắn → setup sinh config mới
         → CircleCI dùng config mới → chạy main workflow linh hoạt
```

---

## 2. Kiến Trúc Hai Giai Đoạn

Dynamic Config chia pipeline thành 2 giai đoạn rõ ràng:

```
┌─────────────────────────────────────────────────────────┐
│  GIAI ĐOẠN 1 — SETUP (pipeline ngắn, chạy nhanh)       │
│                                                         │
│  Job: setup                                             │
│    1. Checkout code                                     │
│    2. Phân tích git diff — sự khác biệt trong git       │
│    3. Quyết định job nào cần chạy                       │
│    4. Tạo file config.yml động                          │
│    5. Gọi continuation API với config mới               │
└─────────────────────────┬───────────────────────────────┘
                          │ truyền generated config
                          ▼
┌─────────────────────────────────────────────────────────┐
│  GIAI ĐOẠN 2 — MAIN WORKFLOW (pipeline thực sự)        │
│                                                         │
│  Chạy các job được sinh ra động:                        │
│    - build-service-a (nếu service A thay đổi)           │
│    - test-service-b  (nếu service B thay đổi)           │
│    - deploy-shared   (nếu shared lib thay đổi)          │
└─────────────────────────────────────────────────────────┘
```

### Continuation API — API Tiếp Nối

CircleCI cung cấp REST API endpoint để giai đoạn setup gọi:

```
POST https://circleci.com/api/v2/pipeline/continue
  Header: Circle-Token: <token>
  Body:
    continuation-key: <key từ environment variable>
    configuration: <nội dung YAML config mới>
    parameters: <pipeline parameters tùy chọn>
```

Biến môi trường `CIRCLE_CONTINUATION_KEY` — Khóa Tiếp Nối được CircleCI tự inject vào setup job khi Dynamic Config được bật.

---

## 3. Bật Dynamic Config

### Bước 1: Bật Trên CircleCI Dashboard

```
Project Settings → Advanced → Enable dynamic config using setup workflows
→ Toggle ON
```

### Bước 2: Đánh Dấu File Config Là Setup

```yaml
# .circleci/config.yml
version: 2.1

# Bắt buộc: khai báo đây là setup config
setup: true

orbs:
  continuation: circleci/continuation@1.0.0

jobs:
  setup:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Quyết định config cần chạy
          command: |
            # Logic của bạn ở đây
            echo "Phân tích thay đổi..."
      - continuation/continue:
          configuration_path: /tmp/generated-config.yml

workflows:
  setup-workflow:
    jobs:
      - setup
```

**Lưu ý quan trọng:** Khi `setup: true` được khai báo, file này chỉ chạy setup workflow. CircleCI sẽ KHÔNG cho phép job thông thường trong file này.

---

## 4. Setup Workflow — Luồng Khởi Tạo

Setup workflow là pipeline ngắn, mục tiêu duy nhất là sinh ra config cho giai đoạn 2.

### Ví Dụ Setup Job Cơ Bản

```yaml
# .circleci/config.yml
version: 2.1
setup: true

orbs:
  continuation: circleci/continuation@1.0.0

jobs:
  generate-config:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout

      - run:
          name: Phân tích git diff để tìm service thay đổi
          command: |
            # Lấy danh sách file thay đổi so với main branch
            CHANGED=$(git diff --name-only origin/main...HEAD)
            echo "File thay đổi:"
            echo "$CHANGED"

            # Xác định service nào bị ảnh hưởng
            if echo "$CHANGED" | grep -q "^services/api/"; then
              echo "BUILD_API=true" >> $BASH_ENV
            fi
            if echo "$CHANGED" | grep -q "^services/worker/"; then
              echo "BUILD_WORKER=true" >> $BASH_ENV
            fi
            if echo "$CHANGED" | grep -q "^libs/shared/"; then
              echo "BUILD_ALL=true" >> $BASH_ENV
            fi

      - run:
          name: Sinh file config động
          command: |
            python3 scripts/generate_config.py > /tmp/generated-config.yml
            echo "=== Config được sinh ra ==="
            cat /tmp/generated-config.yml

      - continuation/continue:
          configuration_path: /tmp/generated-config.yml

workflows:
  setup:
    jobs:
      - generate-config
```

### Script Python Sinh Config

```python
# scripts/generate_config.py
import os
import yaml

def generate_config():
    build_api = os.getenv("BUILD_API") == "true"
    build_worker = os.getenv("BUILD_WORKER") == "true"
    build_all = os.getenv("BUILD_ALL") == "true"

    config = {
        "version": "2.1",
        "jobs": {},
        "workflows": {
            "main": {
                "jobs": []
            }
        }
    }

    # Thêm job build-api nếu cần
    if build_api or build_all:
        config["jobs"]["build-api"] = {
            "docker": [{"image": "cimg/node:20.0"}],
            "steps": [
                "checkout",
                {"run": {"name": "Build API service", "command": "cd services/api && npm run build"}}
            ]
        }
        config["workflows"]["main"]["jobs"].append("build-api")

    # Thêm job build-worker nếu cần
    if build_worker or build_all:
        config["jobs"]["build-worker"] = {
            "docker": [{"image": "cimg/python:3.12"}],
            "steps": [
                "checkout",
                {"run": {"name": "Build Worker service", "command": "cd services/worker && pip install -r requirements.txt && python -m pytest"}}
            ]
        }
        config["workflows"]["main"]["jobs"].append("build-worker")

    # Nếu không có gì thay đổi, chạy job no-op — không làm gì
    if not config["jobs"]:
        config["jobs"]["no-op"] = {
            "docker": [{"image": "cimg/base:stable"}],
            "steps": [
                {"run": {"name": "Không có thay đổi cần build", "command": "echo 'Nothing to build'"}}
            ]
        }
        config["workflows"]["main"]["jobs"].append("no-op")

    print(yaml.dump(config, default_flow_style=False))

if __name__ == "__main__":
    generate_config()
```

---

## 5. Continuation — Tiếp Nối Pipeline

### Dùng Continuation Orb (Cách Được Khuyến Nghị)

```yaml
orbs:
  continuation: circleci/continuation@1.0.0

jobs:
  setup:
    steps:
      # ... sinh config ...
      - continuation/continue:
          configuration_path: /tmp/generated-config.yml
          # Tuỳ chọn: truyền thêm parameters
          parameters: '{"run_integration_tests": true}'
```

### Dùng API Trực Tiếp (Cách Thủ Công)

```bash
# Gọi continuation API trực tiếp bằng curl
curl --request POST \
  --url https://circleci.com/api/v2/pipeline/continue \
  --header "Circle-Token: ${CIRCLE_TOKEN}" \
  --header "Content-Type: application/json" \
  --data "{
    \"continuation-key\": \"${CIRCLE_CONTINUATION_KEY}\",
    \"configuration\": $(cat /tmp/generated-config.yml | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))'),
    \"parameters\": {
      \"environment\": \"staging\"
    }
  }"
```

**Lưu ý:** Continuation chỉ được gọi đúng một lần. Sau khi gọi, `CIRCLE_CONTINUATION_KEY` — Khóa Tiếp Nối bị vô hiệu hóa ngay lập tức.

---

## 6. Ví Dụ Thực Tế

### Ví Dụ 1: Monorepo Với Path-Based Routing

Cấu trúc dự án:
```
monorepo/
├── .circleci/
│   ├── config.yml          ← setup config
│   └── continue-config.yml ← main config template (tùy chọn)
├── services/
│   ├── api/
│   ├── auth/
│   └── notification/
└── libs/
    └── shared/
```

```yaml
# .circleci/config.yml (setup config)
version: 2.1
setup: true

orbs:
  path-filtering: circleci/path-filtering@1.0.0

workflows:
  setup:
    jobs:
      - path-filtering/filter:
          # Mapping: pattern đường dẫn → pipeline parameter
          mapping: |
            services/api/.* build-api true
            services/auth/.* build-auth true
            services/notification/.* build-notification true
            libs/shared/.* build-all true
          base-revision: main
          config-path: .circleci/continue-config.yml
```

```yaml
# .circleci/continue-config.yml (main config với parameters)
version: 2.1

parameters:
  build-api:
    type: boolean
    default: false
  build-auth:
    type: boolean
    default: false
  build-notification:
    type: boolean
    default: false
  build-all:
    type: boolean
    default: false

jobs:
  build-api:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: cd services/api && npm ci && npm run build && npm test

  build-auth:
    docker:
      - image: cimg/go:1.22
    steps:
      - checkout
      - run: cd services/auth && go test ./...

  build-notification:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout
      - run: cd services/notification && pip install -r requirements.txt && pytest

workflows:
  conditional-build:
    jobs:
      - build-api:
          # Chỉ chạy nếu pipeline parameter build-api = true
          when:
            or:
              - << pipeline.parameters.build-api >>
              - << pipeline.parameters.build-all >>

      - build-auth:
          when:
            or:
              - << pipeline.parameters.build-auth >>
              - << pipeline.parameters.build-all >>

      - build-notification:
          when:
            or:
              - << pipeline.parameters.build-notification >>
              - << pipeline.parameters.build-all >>
```

### Ví Dụ 2: Config Động Theo Loại Branch

```bash
#!/bin/bash
# scripts/select-config.sh

BRANCH="${CIRCLE_BRANCH}"
BASE_CONFIG=".circleci/base-config.yml"

if [[ "$BRANCH" == "main" ]]; then
    # Branch main: chạy full test + deploy production
    CONFIG_TEMPLATE=".circleci/configs/production.yml"
elif [[ "$BRANCH" == release/* ]]; then
    # Release branch: chạy full test + deploy staging
    CONFIG_TEMPLATE=".circleci/configs/release.yml"
elif [[ "$BRANCH" == hotfix/* ]]; then
    # Hotfix: fast-track, skip e2e test
    CONFIG_TEMPLATE=".circleci/configs/hotfix.yml"
else
    # Feature branch: chỉ unit test
    CONFIG_TEMPLATE=".circleci/configs/feature.yml"
fi

# Merge base config với template config
python3 scripts/merge_configs.py "$BASE_CONFIG" "$CONFIG_TEMPLATE" > /tmp/final-config.yml
echo "Dùng config: $CONFIG_TEMPLATE"
```

---

## 7. Pipeline Parameters Trong Dynamic Config

Parameters — Tham Số có thể được truyền từ setup sang main workflow:

```yaml
# Trong setup job
- continuation/continue:
    configuration_path: /tmp/config.yml
    parameters: |
      {
        "environment": "staging",
        "run_performance_tests": true,
        "deploy_region": "us-east-1"
      }
```

```yaml
# Trong generated config — config được sinh ra
parameters:
  environment:
    type: string
    default: "development"
  run_performance_tests:
    type: boolean
    default: false
  deploy_region:
    type: string
    default: "us-west-2"

jobs:
  deploy:
    steps:
      - run:
          name: Deploy lên << pipeline.parameters.environment >>
          command: |
            ./deploy.sh \
              --env << pipeline.parameters.environment >> \
              --region << pipeline.parameters.deploy_region >>

workflows:
  main:
    jobs:
      - deploy
      - performance-test:
          requires: [deploy]
          when: << pipeline.parameters.run_performance_tests >>
```

---

## 8. Giới Hạn Và Lưu Ý

### Giới Hạn Kỹ Thuật

| Giới Hạn | Giá Trị |
| --------- | ------- |
| Kích thước config tối đa (generated) | 500 KB |
| Thời gian setup workflow tối đa | Tương tự job thông thường |
| Số lần gọi continuation mỗi pipeline | 1 lần duy nhất |
| Parameters tối đa truyền sang continuation | 100 parameters |

### Lưu Ý Quan Trọng

1. **Setup workflow không thể dùng workspace** — Workspace — Không Gian Làm Việc Chung không hoạt động giữa setup và continuation pipeline.

2. **Artifacts từ setup** — Setup job có thể lưu artifacts như bình thường, nhưng continuation pipeline là pipeline mới độc lập.

3. **Context và secrets** — Contexts — Ngữ Cảnh được áp dụng lại từ đầu trong continuation pipeline, không kế thừa từ setup.

4. **Không thể hủy sau khi gọi continuation** — Một khi `continuation/continue` được gọi, pipeline mới được tạo và không thể rollback.

5. **Debug khó hơn** — Khi có lỗi trong config được sinh động, error message có thể khó đọc hơn config tĩnh.

### Best Practices — Thực Hành Tốt Nhất

```
✅ Nên:
  - Kiểm tra (validate) config sinh ra trước khi gọi continuation
  - Dùng circleci config validate trong setup job
  - Logging — ghi log đầy đủ trong setup để debug dễ
  - Giữ logic sinh config đơn giản, test riêng logic này
  - Có fallback: nếu không có gì thay đổi, sinh config no-op

❌ Tránh:
  - Logic phức tạp trong setup job (giữ setup chạy < 2 phút)
  - Sinh config không được kiểm tra syntax
  - Hardcode values — giá trị cứng trong config được sinh ra
  - Phụ thuộc vào workspace giữa setup và continuation
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Dynamic Config là gì và khi nào cần dùng?**

> Dynamic Config cho phép pipeline CircleCI sinh ra file cấu hình tại runtime thay vì dùng config tĩnh cố định. Cần dùng khi: (1) monorepo cần chỉ build service bị ảnh hưởng, (2) pipeline logic khác nhau tùy theo loại branch hoặc trigger, (3) danh sách job thay đổi động theo dữ liệu bên ngoài như manifest file.

**Q: Continuation API hoạt động như thế nào?**

> Setup pipeline nhận `CIRCLE_CONTINUATION_KEY` — Khóa Tiếp Nối từ CircleCI. Setup job sinh ra config YAML mới rồi POST lên `https://circleci.com/api/v2/pipeline/continue` kèm key đó và config. CircleCI nhận request, xác thực, và khởi tạo pipeline mới với config được cung cấp. Key chỉ dùng được một lần.

**Q: Sự khác biệt giữa Dynamic Config và dùng `when` condition trong workflow thông thường?**

> `when` condition trong workflow vẫn đọc toàn bộ config tĩnh — tất cả job đều được khai báo sẵn, chỉ có điều kiện chạy hay không. Dynamic Config thực sự không khai báo job nào trong file tĩnh — chỉ có setup job. Job thực sự được tạo ra hoàn toàn mới tại runtime. Dynamic Config linh hoạt hơn nhưng phức tạp hơn để debug.

**Q: Làm thế nào để debug khi continuation pipeline bị lỗi config?**

> (1) Thêm bước `cat /tmp/generated-config.yml` và `circleci config validate /tmp/generated-config.yml` trước khi gọi continuation. (2) Lưu generated config làm artifact để xem lại. (3) Test script sinh config cục bộ với `python scripts/generate_config.py > test-config.yml && circleci config validate test-config.yml`.

---

## 🔗 Xem Thêm

- [2-path-filtering.md](2-path-filtering.md) — Path Filtering orb dùng Dynamic Config
- [5-monorepo-strategy.md](5-monorepo-strategy.md) — Chiến lược monorepo tổng thể
- [CircleCI Docs: Dynamic Config](https://circleci.com/docs/dynamic-config/)

---

**Cập Nhật Lần Cuối:** 2026-05-20
