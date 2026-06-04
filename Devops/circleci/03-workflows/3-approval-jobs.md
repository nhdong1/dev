# Approval Jobs — Cổng Duyệt Thủ Công

> **Approval Job** (Công Việc Cần Duyệt Thủ Công) là một loại Job đặc biệt trong CircleCI tạo ra một điểm dừng trong workflow — pipeline sẽ tạm ngưng và chờ cho đến khi người có quyền vào CircleCI UI và bấm nút **Approve** (Duyệt). Đây là cơ chế kiểm soát con người (human gate) trước các bước nguy hiểm như deploy production.

---

## 📚 Mục Lục

1. [Approval Job Là Gì?](#1-approval-job-là-gì)
2. [Cú Pháp Cấu Hình](#2-cú-pháp-cấu-hình)
3. [Luồng Hoạt Động](#3-luồng-hoạt-động)
4. [Ai Có Thể Approve?](#4-ai-có-thể-approve)
5. [Timeout — Hết Thời Gian Chờ](#5-timeout--hết-thời-gian-chờ)
6. [Nhiều Approval Gates](#6-nhiều-approval-gates)
7. [Kết Hợp với Filters](#7-kết-hợp-với-filters)
8. [Mẫu Thực Tế](#8-mẫu-thực-tế)
9. [So Sánh với Các Chiến Lược Khác](#9-so-sánh-với-các-chiến-lược-khác)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Approval Job Là Gì?

Trong một deployment pipeline — đường ống triển khai, thường có những bước cần sự phê duyệt của con người:

- Deploy lên **production** (môi trường sản xuất)
- Chạy **database migration** (di chuyển dữ liệu) không thể rollback
- Xóa tài nguyên cloud (S3 bucket, RDS instance)
- Release tính năng mới ra ngoài

Approval Job giải quyết điều này bằng cách tạo ra một **manual gate** — cổng thủ công trong pipeline.

### Đặc Điểm Của Approval Job

- **Không có executor** — không chạy code, không tốn resource
- **Không có steps** — chỉ chờ con người tương tác
- **`type: approval`** — từ khóa duy nhất để khai báo
- Pipeline **tạm ngưng** tại đây cho đến khi được duyệt hoặc hết timeout
- Có thể **bị từ chối** (Deny/Cancel) — pipeline sẽ fail

---

## 2. Cú Pháp Cấu Hình

### Cú Pháp Tối Giản

```yaml
workflows:
  deploy-pipeline:
    jobs:
      - build
      - test:
          requires: [build]

      # Approval job — không có executor, không có steps
      - approve-deploy:
          type: approval           # Từ khóa bắt buộc
          requires: [test]         # Chờ test xong mới hiện nút Approve

      - deploy-production:
          requires: [approve-deploy]   # Chỉ chạy sau khi được approve
```

### So Sánh: Job Bình Thường vs Approval Job

```yaml
# Job bình thường
jobs:
  build:
    docker:
      - image: cimg/node:20.0    # Cần executor
    steps:                        # Cần steps
      - checkout
      - run: npm run build

# Approval job — trong section workflows, KHÔNG phải jobs
workflows:
  pipeline:
    jobs:
      - approve-production:
          type: approval           # Chỉ cần từ khóa này
          # Không có executor, không có steps, không cần khai báo trong jobs:
```

> **Lưu ý quan trọng:** Approval Job **không được khai báo** trong section `jobs:` của config. Nó chỉ xuất hiện trong `workflows:` và CircleCI tự biết đây là một loại Job đặc biệt.

---

## 3. Luồng Hoạt Động

### Từng Bước Trong Approval Flow

```
1. Code được push → Pipeline trigger

2. Các job chạy bình thường:
   ✅ build    (2 phút)
   ✅ test     (5 phút)

3. Approval job bắt đầu → Pipeline DỪNG lại
   ⏸️  approve-production  ← Đang chờ người duyệt

4. CircleCI gửi thông báo (nếu cấu hình Slack/email)

5. Người có quyền vào CircleCI UI:
   - Xem kết quả test, review code
   - Quyết định: Approve hoặc Deny

6a. Nếu Approve ✅:
    ▶️  deploy-production   (tiếp tục chạy)

6b. Nếu Deny/Cancel ❌:
    ⊘  deploy-production   (bị hủy)
    Pipeline kết thúc với trạng thái "Cancelled"
```

### Giao Diện UI Của CircleCI

Khi pipeline đang chờ approval:
- Workflow hiển thị nút **"On Hold"** (Đang Giữ)
- Job approval hiển thị biểu tượng đồng hồ ⏰
- Người có quyền click vào job → Thấy nút **"Approve"** màu xanh
- Có thể xem thông tin: ai trigger, commit nào, test results ra sao

---

## 4. Ai Có Thể Approve?

Quyền Approve được kiểm soát ở cấp độ **Organization** (Tổ Chức) trong CircleCI.

### Phân Quyền Theo Role — Vai Trò

| Role | Quyền | Mô Tả |
| ---- | ----- | ------ |
| **Admin** | Approve tất cả | Quản trị viên tổ chức |
| **Contributor** | Approve (mặc định) | Thành viên có quyền write vào repo |
| **Viewer** | Không thể Approve | Chỉ xem, không có quyền duyệt |

### Giới Hạn: Ai Không Thể Approve Chính Mình

Một nguyên tắc bảo mật quan trọng: **người trigger pipeline thường không nên tự approve deploy của chính mình** — đây là four-eyes principle (nguyên tắc bốn mắt). Tuy nhiên CircleCI không tự động enforce điều này — cần quy trình tổ chức để đảm bảo.

### Dùng Context để Giới Hạn

```yaml
workflows:
  secure-deploy:
    jobs:
      - approve-production:
          type: approval
          requires: [test]

      - deploy-production:
          requires: [approve-production]
          context: production-deploy-context    # Chỉ những ai có quyền dùng context này
```

Context — Ngữ Cảnh có thể được giới hạn chỉ cho một số nhóm (security groups) trong tổ chức, thêm một lớp bảo vệ bổ sung.

---

## 5. Timeout — Hết Thời Gian Chờ

### Timeout Mặc Định

CircleCI có giới hạn thời gian chờ approval:
- **Mặc định:** Không có timeout tự động — pipeline có thể chờ vô hạn
- **Giới hạn thực tế:** Pipeline được lưu trữ tối đa **90 ngày**

### Thực Hành Tốt: Dùng Scheduled Cleanup

Nếu muốn tự động hủy approval sau một thời gian:
- Thiết lập job cleanup cron để hủy pipeline quá cũ
- Hoặc dùng CircleCI API để tự động cancel

### Thông Báo Nhắc Nhở

```yaml
jobs:
  notify-approval-needed:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Gửi thông báo Slack yêu cầu duyệt
          command: |
            curl -X POST $SLACK_WEBHOOK \
              -H 'Content-type: application/json' \
              --data '{
                "text": "🚀 Pipeline đang chờ duyệt để deploy production!\nCommit: '"$CIRCLE_SHA1"'\nBranch: '"$CIRCLE_BRANCH"'\nLink: '"$CIRCLE_BUILD_URL"'"
              }'

workflows:
  deploy-with-notification:
    jobs:
      - build
      - test:
          requires: [build]

      # Thông báo trước khi chờ duyệt
      - notify-approval-needed:
          requires: [test]
          filters:
            branches:
              only: main

      # Approval gate thực sự
      - approve-production:
          type: approval
          requires: [notify-approval-needed]
          filters:
            branches:
              only: main

      - deploy-production:
          requires: [approve-production]
```

---

## 6. Nhiều Approval Gates

Có thể có nhiều điểm duyệt trong một pipeline — ví dụ: duyệt deploy staging, sau đó lại duyệt deploy production.

```yaml
workflows:
  multi-environment:
    jobs:
      - build
      - test:
          requires: [build]

      # Gate 1: Duyệt deploy staging
      - approve-staging:
          type: approval
          requires: [test]

      - deploy-staging:
          requires: [approve-staging]

      # Gate 2: Duyệt deploy production (sau khi staging OK)
      - approve-production:
          type: approval
          requires: [deploy-staging]

      - deploy-production:
          requires: [approve-production]
          context: production-secrets
```

**Luồng:**
```
build → test → [DUYỆT?] → deploy-staging → [DUYỆT?] → deploy-production
                  ↑                                ↑
            Gate 1: QA lead                Gate 2: Tech Lead/CTO
```

---

## 7. Kết Hợp với Filters

Approval Gate thường chỉ có ý nghĩa trên nhánh production (thường là `main`).

```yaml
workflows:
  ci-cd:
    jobs:
      # Chạy trên mọi nhánh
      - lint
      - test:
          requires: [lint]

      # Chỉ chạy trên main
      - build-production:
          requires: [test]
          filters:
            branches:
              only: main

      # Approval — chỉ trên main, cần build-production xong
      - approve-deploy:
          type: approval
          requires: [build-production]
          filters:
            branches:
              only: main    # Filter bắt buộc phải lặp lại ở đây

      # Deploy — chỉ trên main, cần approval
      - deploy-production:
          requires: [approve-deploy]
          context: aws-production
          filters:
            branches:
              only: main    # Filter bắt buộc phải lặp lại ở đây
```

> **Quy tắc Filter:** Khi một job có `filters`, tất cả các job phụ thuộc sau đó cũng phải có cùng `filters`. Nếu không, job sau sẽ không bao giờ chạy vì job trước không chạy trên nhánh đó.

---

## 8. Mẫu Thực Tế

### Mẫu A: Full Deployment Pipeline với Approval

```yaml
version: 2.1

orbs:
  slack: circleci/slack@4.12.0
  aws-cli: circleci/aws-cli@4.1.0

executors:
  default:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

jobs:
  install-deps:
    executor: default
    steps:
      - checkout
      - restore_cache:
          keys: [npm-{{ checksum "package-lock.json" }}]
      - run: npm ci
      - save_cache:
          key: npm-{{ checksum "package-lock.json" }}
          paths: [~/.npm]
      - persist_to_workspace:
          root: .
          paths: [node_modules/]

  lint:
    executor: default
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run lint

  test:
    executor: default
    parallelism: 3
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run:
          command: |
            circleci tests glob "**/*.test.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --ci
      - store_test_results:
          path: reports/

  build:
    executor: default
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy-staging:
    executor: default
    steps:
      - attach_workspace: {at: .}
      - aws-cli/setup
      - run:
          name: Deploy lên staging
          command: aws s3 sync dist/ s3://my-app-staging --delete
      - slack/notify:
          event: pass
          template: basic_success_1
          channel: "#deployments"

  smoke-test-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Smoke test staging environment
          command: |
            # Smoke test — Kiểm Thử Khói: kiểm tra nhanh xem app có hoạt động không
            curl -f https://staging.myapp.com/health || exit 1
            curl -f https://staging.myapp.com/api/version || exit 1

  deploy-production:
    executor: default
    steps:
      - attach_workspace: {at: .}
      - aws-cli/setup
      - run:
          name: Deploy lên production
          command: |
            aws s3 sync dist/ s3://my-app-production --delete
            aws cloudfront create-invalidation \
              --distribution-id $CF_DISTRIBUTION_ID \
              --paths "/*"
      - slack/notify:
          event: pass
          custom: |
            {
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "✅ *Production deployed successfully!*\nCommit: `$CIRCLE_SHA1`\nBy: $CIRCLE_USERNAME"
                }
              }]
            }

  rollback-production:
    executor: default
    steps:
      - run:
          name: Rollback production nếu cần
          command: ./scripts/rollback.sh production

workflows:
  full-pipeline:
    jobs:
      # Phase 1: Quality checks — Kiểm tra chất lượng
      - install-deps
      - lint:
          requires: [install-deps]
      - test:
          requires: [install-deps]
      - build:
          requires: [lint, test]

      # Phase 2: Deploy staging (tự động)
      - deploy-staging:
          requires: [build]
          context: aws-staging
          filters:
            branches:
              only: main

      # Phase 3: Smoke test staging
      - smoke-test-staging:
          requires: [deploy-staging]
          filters:
            branches:
              only: main

      # Phase 4: Manual approval — Cổng duyệt thủ công
      - approve-production:
          type: approval
          requires: [smoke-test-staging]
          filters:
            branches:
              only: main

      # Phase 5: Deploy production
      - deploy-production:
          requires: [approve-production]
          context: aws-production    # Context riêng cho production
          filters:
            branches:
              only: main

      # Phase 6 (tùy chọn): Rollback gate nếu production có vấn đề
      - approve-rollback:
          type: approval
          requires: [deploy-production]
          filters:
            branches:
              only: main

      - rollback-production:
          requires: [approve-rollback]
          context: aws-production
          filters:
            branches:
              only: main
```

### Mẫu B: Database Migration Approval

```yaml
version: 2.1

jobs:
  run-migration:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - run:
          name: Kiểm tra migration files
          command: python manage.py showmigrations --list
      - run:
          name: Chạy database migration
          command: python manage.py migrate --verbosity 2
      - run:
          name: Verify sau migration
          command: python manage.py check --database default

  notify-migration:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Hiển thị các migration sẽ chạy
          command: python manage.py showmigrations --plan
      - run:
          name: Gửi thông báo cần review
          command: |
            echo "Migration plan đã được post lên Slack"
            # curl ... gửi migration plan lên Slack

workflows:
  database-deploy:
    jobs:
      - notify-migration:
          filters:
            branches:
              only: main

      # Approval trước khi chạy migration không thể rollback
      - approve-migration:
          type: approval
          requires: [notify-migration]
          filters:
            branches:
              only: main

      - run-migration:
          requires: [approve-migration]
          context: production-db
          filters:
            branches:
              only: main
```

---

## 9. So Sánh với Các Chiến Lược Khác

| Phương Pháp | Mô Tả | Ưu Điểm | Nhược Điểm |
| ----------- | ------ | -------- | ---------- |
| **Approval Job** | Human gate trong CircleCI UI | Đơn giản, không cần công cụ thêm | Phải có người online |
| **Branch Protection** | Require PR review trước khi merge | Enforce code review | Không kiểm soát deployment timing |
| **External Tool** (OpsGenie, PagerDuty) | Approval qua công cụ ngoài | Nhiều tính năng hơn | Cấu hình phức tạp hơn |
| **Time-based Deploy** | Chỉ deploy trong giờ hành chính | Tự động | Không linh hoạt cho incidents |
| **Automated Rollback** | Tự động rollback nếu metric xấu | Không cần con người | Cần monitoring tốt |

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Approval Job trong CircleCI là gì và khi nào dùng?**

> Approval Job là Job đặc biệt với `type: approval` không chạy code mà tạo ra một điểm dừng trong pipeline — yêu cầu người có quyền vào CircleCI UI và bấm Approve để tiếp tục. Dùng khi cần kiểm soát con người trước các bước nguy hiểm: deploy production, database migration không reversible, hoặc xóa tài nguyên cloud quan trọng.

**Q: Làm thế nào để khai báo Approval Job trong config.yml?**

> Approval Job **không khai báo** trong section `jobs:` — chỉ khai báo trong `workflows:` với `type: approval`. Ví dụ: `- approve-deploy: type: approval requires: [test]`. CircleCI tự nhận biết đây là approval gate và không tìm executor hay steps cho nó.

**Q: Ai có thể Approve một Approval Job?**

> Mặc định, tất cả thành viên có quyền Contributor trở lên trong tổ chức CircleCI có thể Approve. Để kiểm soát chặt hơn, có thể kết hợp với Context có security groups — chỉ những người trong nhóm được phép dùng Context mới có thể trigger job sau approval.

**Q: Điều gì xảy ra nếu Approval Job bị Cancel hoặc Deny?**

> Pipeline kết thúc với trạng thái "Cancelled" — tất cả các Job phụ thuộc vào Approval Job bị hủy và không chạy. Không có cách retry Approval Job trực tiếp — phải trigger lại toàn bộ pipeline (push code mới hoặc rerun from start).

---

## 📖 Đọc Tiếp

- [4-scheduled-pipelines.md](4-scheduled-pipelines.md) — Kích hoạt pipeline tự động theo lịch
- [5-branch-tag-filters.md](5-branch-tag-filters.md) — Giới hạn Approval chỉ trên main/production
- [06-security/1-contexts.md](../06-security/1-contexts.md) — Dùng Context để bảo mật production deploy

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
