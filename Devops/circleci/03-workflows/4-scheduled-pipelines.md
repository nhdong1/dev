# Scheduled Pipelines — Pipeline Kích Hoạt Theo Lịch

> **Scheduled Pipelines** (Pipeline Kích Hoạt Theo Lịch) là tính năng cho phép CircleCI tự động trigger — kích hoạt pipeline theo lịch định sẵn bằng cú pháp **cron** (Cron Expression — Biểu Thức Lập Lịch), không cần sự kiện push code hay tương tác của người dùng.

---

## 📚 Mục Lục

1. [Tại Sao Cần Scheduled Pipelines?](#1-tại-sao-cần-scheduled-pipelines)
2. [Cú Pháp Cron — Biểu Thức Thời Gian](#2-cú-pháp-cron--biểu-thức-thời-gian)
3. [Cấu Hình Scheduled Pipelines](#3-cấu-hình-scheduled-pipelines)
4. [Phân Biệt Scheduled vs Code-Triggered](#4-phân-biệt-scheduled-vs-code-triggered)
5. [Triggers — Nguồn Kích Hoạt](#5-triggers--nguồn-kích-hoạt)
6. [Mẫu Use Cases Phổ Biến](#6-mẫu-use-cases-phổ-biến)
7. [Quản Lý Scheduled Pipelines](#7-quản-lý-scheduled-pipelines)
8. [Lỗi Thường Gặp](#8-lỗi-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần Scheduled Pipelines?

Không phải mọi pipeline đều cần trigger từ code push. Có nhiều trường hợp cần chạy tự động theo lịch cố định:

### Các Use Case Phổ Biến

| Use Case | Mô Tả | Tần Suất |
| -------- | ------ | -------- |
| **Nightly Build** — Build Hằng Đêm | Chạy full test suite vào ban đêm khi server rảnh | Hàng đêm 2:00 AM |
| **Security Scan** — Quét Bảo Mật | Kiểm tra CVE mới trong dependencies | Hàng ngày |
| **Dependency Audit** — Kiểm Tra Phụ Thuộc | `npm audit`, `snyk test` | Hàng ngày hoặc hàng tuần |
| **Performance Test** — Kiểm Thử Hiệu Năng | Chạy load test khi traffic thấp | Hàng tuần |
| **Database Backup** — Sao Lưu CSDL | Tạo snapshot định kỳ | Hàng ngày |
| **Report Generation** — Tạo Báo Cáo | Tổng hợp metrics và gửi email | Hàng tuần/tháng |
| **Cache Warmup** — Làm Nóng Cache | Xây dựng cache trước giờ cao điểm | Mỗi sáng 7:00 AM |
| **Cleanup Jobs** — Dọn Dẹp | Xóa Docker images cũ, log files | Hàng tuần |

---

## 2. Cú Pháp Cron — Biểu Thức Thời Gian

Cron sử dụng **5 trường** (fields) để định nghĩa lịch chạy, tất cả theo múi giờ **UTC** — Coordinated Universal Time (Giờ Phối Hợp Quốc Tế).

### Cấu Trúc

```
┌───────── Phút (0–59)
│ ┌─────── Giờ (0–23)
│ │ ┌───── Ngày trong tháng (1–31)
│ │ │ ┌─── Tháng (1–12)
│ │ │ │ ┌─ Ngày trong tuần (0–6, 0=Chủ Nhật, 7=Chủ Nhật)
│ │ │ │ │
* * * * *
```

### Ký Hiệu Đặc Biệt

| Ký Hiệu | Ý Nghĩa | Ví Dụ |
| ------- | ------- | ----- |
| `*` | Mọi giá trị | `* * * * *` — mỗi phút |
| `,` | Danh sách giá trị | `0 9,18 * * *` — 9 AM và 6 PM |
| `-` | Khoảng giá trị | `0 9-17 * * *` — mỗi giờ từ 9 đến 17 |
| `/` | Bước nhảy (step) | `0 */4 * * *` — mỗi 4 giờ |

### Ví Dụ Cron Phổ Biến

```bash
# Mỗi ngày lúc 2:00 AM UTC
0 2 * * *

# Mỗi ngày lúc 9:00 AM UTC (16:00 ICT — Indochina Time — Giờ Đông Dương)
0 9 * * *

# Thứ Hai đến Thứ Sáu lúc 8:00 AM UTC (ngày làm việc)
0 8 * * 1-5

# Mỗi Chủ Nhật lúc 1:00 AM UTC
0 1 * * 0

# Ngày 1 hàng tháng lúc 0:00 AM UTC
0 0 1 * *

# Mỗi 6 giờ
0 */6 * * *

# 9:30 AM thứ Hai đến thứ Sáu
30 9 * * 1-5

# Hai lần mỗi tuần: thứ Tư và thứ Sáu lúc 3:00 AM
0 3 * * 3,5
```

> **Múi Giờ:** CircleCI sử dụng **UTC** cho tất cả cron expressions. Để chạy lúc 8:00 AM ICT (GMT+7), cần đặt cron là `0 1 * * *` (1:00 AM UTC = 8:00 AM ICT).

### Công Cụ Kiểm Tra Cron

Dùng [crontab.guru](https://crontab.guru/) để kiểm tra cú pháp cron và xem giải thích bằng ngôn ngữ tự nhiên.

---

## 3. Cấu Hình Scheduled Pipelines

CircleCI cung cấp **hai cách** để cấu hình scheduled pipelines:

### Cách 1: Project Settings — Giao Diện Web (Khuyến Nghị)

1. Vào **CircleCI App** → Chọn Project
2. Vào **Project Settings** (Cài Đặt Dự Án)
3. Chọn tab **Triggers** — Nguồn Kích Hoạt
4. Bấm **Add Trigger** → Chọn **Scheduled**
5. Điền thông tin:
   - **Name**: Tên schedule (ví dụ: `nightly-build`)
   - **Cron**: Biểu thức cron
   - **Branch**: Nhánh nào sẽ chạy
   - **Pipeline Parameters**: Tham số tùy chọn

**Ưu điểm:** Không cần sửa config file, dễ bật/tắt từ UI, dễ xem danh sách tất cả schedules.

### Cách 2: `.circleci/config.yml` (Cách Cũ — Đã Lỗi Thời)

```yaml
# ⚠️ DEPRECATED — Không nên dùng từ CircleCI 2.1 trở đi
# Chỉ dùng với version: 2.0

workflows:
  nightly:
    triggers:
      - schedule:
          cron: "0 2 * * *"
          filters:
            branches:
              only:
                - main
    jobs:
      - nightly-test
      - security-scan
```

> **Khuyến nghị:** Dùng **Project Settings → Triggers** thay vì `triggers` trong config.yml vì cách cũ không hỗ trợ pipeline parameters và ít linh hoạt hơn.

---

## 4. Phân Biệt Scheduled vs Code-Triggered

Khi dùng scheduled pipelines, thường cần chạy **workflow khác** hoặc **job khác** so với code push thông thường. CircleCI cung cấp **Pipeline Values** — Giá Trị Pipeline để phân biệt.

### Pipeline Values Quan Trọng

```yaml
# Các biến môi trường tự động của CircleCI
CIRCLE_PIPELINE_TRIGGER_SOURCE    # "scheduled" hoặc "push" hoặc "api"
PIPELINE_PARAMETERS_*             # Các parameter tùy chỉnh
```

### Phân Biệt Bằng Pipeline Parameters

Khi tạo scheduled trigger qua UI, có thể thêm **Pipeline Parameters** (Tham Số Pipeline) tùy chỉnh:

```yaml
# .circleci/config.yml
version: 2.1

# Khai báo parameters ở cấp pipeline
parameters:
  run-schedule:          # Tham số để phân biệt scheduled vs push
    type: boolean
    default: false
  schedule-type:         # Loại schedule: nightly, weekly, monthly
    type: string
    default: ""

jobs:
  nightly-full-test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - run:
          name: Chạy toàn bộ test suite (bao gồm integration và e2e)
          command: npm run test:all

  quick-test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - run: npm run test:unit

  security-audit:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm audit --audit-level=high
      - run: npx snyk test

  weekly-report:
    docker:
      - image: cimg/python:3.11
    steps:
      - run: python scripts/generate_weekly_report.py

workflows:
  # Workflow bình thường khi push code
  push-workflow:
    when:
      not: << pipeline.parameters.run-schedule >>   # Chỉ chạy khi KHÔNG phải schedule
    jobs:
      - quick-test

  # Workflow cho nightly schedule
  nightly-workflow:
    when:
      and:
        - << pipeline.parameters.run-schedule >>
        - equal: [nightly, << pipeline.parameters.schedule-type >>]
    jobs:
      - nightly-full-test
      - security-audit

  # Workflow cho weekly schedule
  weekly-workflow:
    when:
      and:
        - << pipeline.parameters.run-schedule >>
        - equal: [weekly, << pipeline.parameters.schedule-type >>]
    jobs:
      - weekly-report
```

**Cấu hình trong UI (Project Settings → Triggers):**
```
Trigger 1 — Nightly:
  Cron: 0 2 * * *
  Branch: main
  Parameters:
    run-schedule: true
    schedule-type: "nightly"

Trigger 2 — Weekly:
  Cron: 0 3 * * 0
  Branch: main
  Parameters:
    run-schedule: true
    schedule-type: "weekly"
```

---

## 5. Triggers — Nguồn Kích Hoạt

CircleCI hỗ trợ 3 loại trigger:

### 1. Push Trigger (Mặc Định)

```
Developer push code → GitHub/Bitbucket webhook → CircleCI trigger pipeline
```

### 2. Scheduled Trigger

```
Thời gian định sẵn (cron) → CircleCI scheduler → Trigger pipeline
```

### 3. API Trigger — Kích Hoạt Qua API

```bash
# Trigger pipeline thủ công qua CircleCI API
curl --request POST \
  --url https://circleci.com/api/v2/project/github/my-org/my-repo/pipeline \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "content-type: application/json" \
  --data '{
    "branch": "main",
    "parameters": {
      "run-schedule": true,
      "schedule-type": "manual"
    }
  }'
```

---

## 6. Mẫu Use Cases Phổ Biến

### Mẫu A: Nightly Build — Build Hằng Đêm

```yaml
version: 2.1

parameters:
  is-scheduled:
    type: boolean
    default: false

jobs:
  full-regression-test:
    docker:
      - image: cimg/node:20.0
    resource_class: large            # Dùng resource lớn hơn ban đêm
    parallelism: 8
    steps:
      - checkout
      - run: npm ci
      - run:
          name: Chạy full regression test suite
          command: |
            circleci tests glob "**/*.test.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --ci --coverage
      - store_test_results:
          path: reports/

  performance-benchmark:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run: npm ci && npm run build
      - run:
          name: Chạy performance benchmark
          command: npm run benchmark -- --iterations=1000

  notify-nightly-result:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Gửi báo cáo nightly build lên Slack
          command: |
            curl -X POST $SLACK_WEBHOOK \
              --data-urlencode "payload={\"text\": \"🌙 Nightly build hoàn thành: $CIRCLE_BUILD_URL\"}"

workflows:
  nightly-build:
    when: << pipeline.parameters.is-scheduled >>
    jobs:
      - full-regression-test
      - performance-benchmark:
          requires: [full-regression-test]
      - notify-nightly-result:
          requires: [performance-benchmark]

  regular-ci:
    when:
      not: << pipeline.parameters.is-scheduled >>
    jobs:
      - quick-test    # Job đơn giản hơn khi push code thường
```

### Mẫu B: Security Scanning Hàng Ngày

```yaml
version: 2.1

parameters:
  security-scan:
    type: boolean
    default: false

jobs:
  dependency-audit:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run:
          name: Kiểm tra vulnerabilities trong npm packages
          command: npm audit --audit-level=moderate
      - run:
          name: Snyk security test
          command: npx snyk test --severity-threshold=high

  docker-image-scan:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run:
          name: Build và scan Docker image
          command: |
            docker build -t my-app:scan .
            # Trivy — công cụ quét lỗ hổng bảo mật container
            trivy image --exit-code 1 \
              --severity HIGH,CRITICAL \
              --format template \
              --template "@contrib/html.tpl" \
              -o trivy-report.html \
              my-app:scan
      - store_artifacts:
          path: trivy-report.html
          destination: security-reports

  sast-scan:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - run:
          name: SAST — Static Application Security Testing — Kiểm Thử Bảo Mật Tĩnh
          command: |
            pip install bandit semgrep
            bandit -r src/ -f json -o bandit-report.json || true
            semgrep --config=auto src/ --json > semgrep-report.json || true
      - store_artifacts:
          path: bandit-report.json
      - store_artifacts:
          path: semgrep-report.json

  create-security-ticket:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Tạo Jira ticket nếu có lỗ hổng nghiêm trọng
          command: ./scripts/create-security-ticket.sh

workflows:
  daily-security-scan:
    when: << pipeline.parameters.security-scan >>
    jobs:
      - dependency-audit
      - docker-image-scan
      - sast-scan
      - create-security-ticket:
          requires:
            - dependency-audit
            - docker-image-scan
            - sast-scan
```

### Mẫu C: Weekly Report — Báo Cáo Hàng Tuần

```yaml
version: 2.1

parameters:
  weekly-report:
    type: boolean
    default: false

jobs:
  generate-coverage-report:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - run: pip install pandas matplotlib jinja2
      - run:
          name: Tạo báo cáo coverage tuần
          command: python scripts/generate_coverage_report.py
      - store_artifacts:
          path: reports/weekly-coverage.html

  generate-build-metrics:
    docker:
      - image: cimg/python:3.11
    steps:
      - run:
          name: Lấy và tổng hợp CircleCI Insights metrics
          command: |
            # Gọi CircleCI Insights API — API Thống Kê
            python scripts/fetch_circleci_insights.py \
              --project-slug github/my-org/my-repo \
              --branch main \
              --reporting-window last-90-days

  send-weekly-email:
    docker:
      - image: cimg/python:3.11
    steps:
      - run:
          name: Gửi weekly report email
          command: python scripts/send_weekly_report.py

workflows:
  weekly-reporting:
    when: << pipeline.parameters.weekly-report >>
    jobs:
      - generate-coverage-report
      - generate-build-metrics
      - send-weekly-email:
          requires:
            - generate-coverage-report
            - generate-build-metrics
```

---

## 7. Quản Lý Scheduled Pipelines

### Xem Danh Sách Schedules

```bash
# CircleCI API v2 — Lấy danh sách tất cả schedules
curl --request GET \
  --url "https://circleci.com/api/v2/project/github/my-org/my-repo/schedule" \
  --header "Circle-Token: $CIRCLECI_TOKEN"
```

### Tạm Dừng Schedule

Trong CircleCI UI:
1. Project Settings → Triggers
2. Tìm schedule muốn tạm dừng
3. Toggle "Active" → Off

Qua API:
```bash
curl --request PATCH \
  --url "https://circleci.com/api/v2/schedule/SCHEDULE_ID" \
  --header "Circle-Token: $CIRCLECI_TOKEN" \
  --header "content-type: application/json" \
  --data '{"timetable": {"per-hour": 0}}'   # Đặt tần suất về 0
```

### Theo Dõi Scheduled Pipeline Runs

```bash
# Xem lịch sử pipeline runs của một schedule
curl --request GET \
  --url "https://circleci.com/api/v2/project/github/my-org/my-repo/pipeline?branch=main" \
  --header "Circle-Token: $CIRCLECI_TOKEN" | \
  jq '.items[] | select(.trigger.type == "scheduled_pipeline")'
```

---

## 8. Lỗi Thường Gặp

### Lỗi 1: Cron Không Trigger

**Nguyên nhân phổ biến:**
- Cron expression sai cú pháp
- Pipeline đang bị "Suspend" — Tạm Dừng (chạy hết credits)
- Branch được chỉ định không tồn tại
- Schedule bị disabled

**Kiểm tra:**
```bash
# Kiểm tra schedule configuration
curl --request GET \
  --url "https://circleci.com/api/v2/schedule/SCHEDULE_ID" \
  --header "Circle-Token: $CIRCLECI_TOKEN" | jq .
```

### Lỗi 2: Scheduled và Push Workflow Chạy Cùng Lúc

**Vấn đề:** Không phân biệt được hai loại trigger → cả hai workflow đều chạy.

**Giải pháp:** Dùng pipeline parameters để phân biệt:

```yaml
# Đặt parameter run-schedule=true cho scheduled trigger trong UI
workflows:
  push-only:
    when:
      not: << pipeline.parameters.run-schedule >>
    jobs: [quick-test]

  scheduled-only:
    when: << pipeline.parameters.run-schedule >>
    jobs: [full-test, security-scan]
```

### Lỗi 3: Quên Múi Giờ UTC

**Vấn đề:** Muốn chạy lúc 8 AM ICT nhưng đặt cron `0 8 * * *` → thực ra chạy lúc 3 PM ICT.

**Giải pháp:** Luôn chuyển đổi sang UTC trước:
```
ICT (GMT+7) 8:00 AM = UTC 1:00 AM → cron: 0 1 * * *
ICT (GMT+7) 9:00 PM = UTC 2:00 PM → cron: 0 14 * * *
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Scheduled Pipelines trong CircleCI dùng để làm gì?**

> Scheduled Pipelines cho phép chạy pipeline tự động theo lịch định sẵn bằng cú pháp cron — không cần push code. Dùng cho nightly build (chạy full test suite ban đêm), daily security scan (quét CVE mới trong dependencies hàng ngày), weekly report generation (tổng hợp metrics hàng tuần), database backup định kỳ, và các tác vụ maintenance.

**Q: Có mấy cách cấu hình Scheduled Pipelines?**

> Có hai cách: (1) Qua **Project Settings → Triggers** trong UI — cách hiện đại, hỗ trợ pipeline parameters, dễ bật/tắt; (2) Trong `config.yml` với `triggers.schedule` — cách cũ, đã deprecated từ khi CircleCI ra tính năng scheduled pipelines mới. Nên dùng cách 1.

**Q: Làm thế nào để một workflow chỉ chạy khi được schedule, không chạy khi push code?**

> Dùng **pipeline parameters**: tạo parameter `is-scheduled: boolean` với default `false`. Khi cấu hình schedule trong UI, đặt parameter này là `true`. Trong config, dùng `when: << pipeline.parameters.is-scheduled >>` cho scheduled workflow và `when: not: << pipeline.parameters.is-scheduled >>` cho push workflow.

**Q: Múi giờ nào CircleCI dùng cho cron expressions?**

> CircleCI luôn dùng **UTC** (Coordinated Universal Time). Để chạy lúc 8:00 AM ICT (GMT+7), cần đặt cron là `0 1 * * *` (vì 8 AM ICT = 1 AM UTC).

---

## 📖 Đọc Tiếp

- [5-branch-tag-filters.md](5-branch-tag-filters.md) — Kết hợp schedule với branch filters
- [09-advanced/1-dynamic-config.md](../09-advanced/1-dynamic-config.md) — Dynamic Config để tạo workflow linh hoạt hơn
- [08-monitoring/1-pipeline-insights.md](../08-monitoring/1-pipeline-insights.md) — Theo dõi scheduled pipeline performance

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
