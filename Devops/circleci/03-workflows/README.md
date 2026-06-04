# 03 — Workflows — Luồng Công Việc trong CircleCI

> Module này bao gồm toàn bộ kiến thức về Workflow — Luồng Công Việc trong CircleCI: từ sequential (tuần tự), parallel (song song), approval gate (cổng duyệt thủ công), scheduled pipelines (kích hoạt theo lịch) đến branch/tag filters (bộ lọc nhánh và nhãn).

---

## 📚 Mục Lục Module

| File | Nội Dung | Mức Độ |
| ---- | -------- | ------ |
| [1-sequential-workflow.md](1-sequential-workflow.md) | `requires`, chuỗi phụ thuộc job | Cơ Bản |
| [2-parallel-workflow.md](2-parallel-workflow.md) | Fan-out/Fan-in — Phân Tỏa/Tập Hợp, tăng tốc build | Cơ Bản |
| [3-approval-jobs.md](3-approval-jobs.md) | Manual gate — Cổng Duyệt Thủ Công trước khi deploy | Trung Cấp |
| [4-scheduled-pipelines.md](4-scheduled-pipelines.md) | Cron triggers — Kích Hoạt Theo Lịch | Trung Cấp |
| [5-branch-tag-filters.md](5-branch-tag-filters.md) | Bộ lọc nhánh, tag, regex trong workflow | Trung Cấp |

---

## 🎯 Workflow là Gì?

**Workflow** (Luồng Công Việc) là tập hợp các quy tắc xác định cách các **Job** (Công Việc) chạy — thứ tự, điều kiện, và mối quan hệ phụ thuộc lẫn nhau.

### Vị Trí trong Kiến Trúc CircleCI

```
Pipeline — Đường Ống CI/CD (trigger: push code, API call, schedule)
└── Workflow — Luồng Công Việc (xác định thứ tự & điều kiện)
    ├── Job A ─── Job B ─── Job C     (tuần tự — sequential)
    ├── Job D ─┐
    │          ├── Job F              (song song — parallel)
    └── Job E ─┘
```

**Pipeline** — Đường Ống CI/CD: toàn bộ quá trình từ trigger đến kết quả cuối.
**Workflow**: điều phối (orchestrate) thứ tự và điều kiện chạy các Job.
**Job** — Công Việc: tập hợp các Step chạy trên một Executor — Môi Trường Thực Thi.

---

## 🏗️ Cấu Trúc Cơ Bản của Workflow

```yaml
# .circleci/config.yml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - run: npm run build

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - run: npm test

  deploy:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: ./scripts/deploy.sh

workflows:              # Bắt buộc phải có section này
  ci-cd:                # Tên workflow — tùy đặt
    jobs:
      - build           # Job đầu tiên — không có điều kiện gì
      - test:
          requires:
            - build     # test chỉ chạy sau khi build thành công
      - deploy:
          requires:
            - test      # deploy chỉ chạy sau khi test xanh
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### 1. `requires` — Phụ Thuộc Giữa Các Job

Xác định Job nào phải **hoàn thành thành công** trước khi Job hiện tại được phép chạy.

```yaml
workflows:
  pipeline:
    jobs:
      - lint
      - unit-test:
          requires:
            - lint        # unit-test chờ lint xong
      - integration-test:
          requires:
            - unit-test   # integration-test chờ unit-test xong
      - deploy:
          requires:
            - integration-test
```

> **Lưu ý:** Nếu một Job trong `requires` thất bại (fail), tất cả Job phụ thuộc vào nó đều bị hủy (cancelled). Đây là cơ chế **fail-fast** — thất bại nhanh để tiết kiệm tài nguyên.

### 2. Parallel Execution — Thực Thi Song Song

Các Job **không có** quan hệ `requires` với nhau sẽ chạy **đồng thời** — đây là mặc định của CircleCI.

```yaml
workflows:
  parallel:
    jobs:
      - lint            # ┐
      - unit-test       # ├── Ba job này chạy cùng lúc
      - security-scan   # ┘
      - build:
          requires:
            - lint        # build chờ cả 3 job trên xong
            - unit-test
            - security-scan
```

### 3. `type: approval` — Cổng Duyệt Thủ Công

Tạo một điểm dừng — người có quyền phải vào CircleCI UI và bấm **Approve** thì pipeline mới tiếp tục.

```yaml
workflows:
  deploy-pipeline:
    jobs:
      - build
      - test:
          requires: [build]
      - approve-production:
          type: approval        # Job đặc biệt — chỉ chờ người duyệt
          requires: [test]
      - deploy-production:
          requires: [approve-production]
```

### 4. `filters` — Bộ Lọc Điều Kiện Chạy

Xác định khi nào một Job được phép chạy dựa trên tên nhánh (branch) hoặc nhãn (tag).

```yaml
workflows:
  pipeline:
    jobs:
      - test:
          filters:
            branches:
              only: /.*/          # Chạy trên mọi nhánh
      - deploy:
          filters:
            branches:
              only: main          # Chỉ chạy trên nhánh main
          requires: [test]
```

---

## 📊 Các Mẫu Workflow Phổ Biến

### Mẫu 1: CI Đơn Giản (Feature Branch)

```
push code → lint → test → (kết thúc nếu không phải main)
```

### Mẫu 2: CI/CD Hoàn Chỉnh

```
push code → lint ─┐
                  ├── build → approve → deploy-prod
           test  ─┘
```

### Mẫu 3: Multi-Environment Deployment

```
push main → test → deploy-staging → approve → deploy-production
```

### Mẫu 4: Scheduled Nightly Build

```
cron 2:00 AM → full-regression-test → performance-test → notify
```

---

## 🗺️ Sơ Đồ Quyết Định — Chọn Loại Workflow

```
Bạn cần gì?
│
├── Chạy job theo thứ tự có phụ thuộc?
│   └─► Sequential Workflow → [1-sequential-workflow.md]
│
├── Chạy nhiều job song song để tăng tốc?
│   └─► Parallel / Fan-out/Fan-in → [2-parallel-workflow.md]
│
├── Cần người duyệt trước khi deploy production?
│   └─► Approval Job → [3-approval-jobs.md]
│
├── Chạy pipeline tự động theo lịch (nightly build, weekly report)?
│   └─► Scheduled Pipelines → [4-scheduled-pipelines.md]
│
└── Chỉ deploy khi push lên nhánh main hoặc tag v*?
    └─► Branch/Tag Filters → [5-branch-tag-filters.md]
```

---

## ⚙️ Cấu Hình Workflow Đầy Đủ — Production-Ready

```yaml
version: 2.1

orbs:
  slack: circleci/slack@4.12.0

executors:
  default:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

jobs:
  lint:
    executor: default
    steps:
      - checkout
      - run: npm ci
      - run: npm run lint

  unit-test:
    executor: default
    parallelism: 4
    steps:
      - checkout
      - run: npm ci
      - run:
          command: |
            circleci tests glob "src/**/*.test.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --ci

  build:
    executor: default
    steps:
      - checkout
      - run: npm ci && npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy-staging:
    executor: default
    steps:
      - attach_workspace:
          at: .
      - run: ./scripts/deploy.sh staging

  approve-production:
    # Job đặc biệt — không có executor, không có steps
    # CircleCI tự động tạo nút Approve trong UI
    type: approval

  deploy-production:
    executor: default
    steps:
      - attach_workspace:
          at: .
      - run: ./scripts/deploy.sh production
      - slack/notify:
          event: pass
          template: basic_success_1

workflows:
  main-pipeline:
    jobs:
      # Phase 1: Kiểm tra chất lượng — chạy song song
      - lint
      - unit-test

      # Phase 2: Build — chờ phase 1 xong
      - build:
          requires:
            - lint
            - unit-test

      # Phase 3: Deploy staging — chỉ từ nhánh main
      - deploy-staging:
          requires: [build]
          filters:
            branches:
              only: main

      # Phase 4: Approval gate — cổng duyệt trước production
      - approve-production:
          type: approval
          requires: [deploy-staging]
          filters:
            branches:
              only: main

      # Phase 5: Deploy production — sau khi được duyệt
      - deploy-production:
          requires: [approve-production]
          context: production-secrets    # Dùng Context để lấy secrets
          filters:
            branches:
              only: main

  # Workflow riêng cho feature branches — chỉ test, không deploy
  feature-branch:
    jobs:
      - lint:
          filters:
            branches:
              ignore: main
      - unit-test:
          requires: [lint]
          filters:
            branches:
              ignore: main
```

---

## 🔍 Lỗi Thường Gặp & Cách Xử Lý

| Lỗi | Nguyên Nhân | Cách Xử Lý |
| --- | ----------- | ----------- |
| Job bị `blocked` mãi | Job trong `requires` fail hoặc chưa chạy | Kiểm tra log job phía trước |
| Approval job không xuất hiện | Thiếu `type: approval` | Thêm đúng vào config |
| Deploy chạy cả trên feature branch | Thiếu `filters` | Thêm `filters.branches.only: main` |
| Scheduled pipeline không trigger | Cấu hình cron sai hoặc pipeline bị suspend | Kiểm tra Project Settings → Triggers |
| Regex filter không match | Regex sai cú pháp | Dùng `/^pattern$/` và test trước |

---

## 🎯 Tóm Tắt Nhanh cho Phỏng Vấn

**Workflow là gì?**
> Workflow định nghĩa cách các Job phối hợp với nhau: thứ tự chạy, điều kiện kích hoạt, và điều kiện cho phép chạy. Không có Workflow, các Job không chạy được.

**Sự khác biệt giữa Workflow và Pipeline?**
> Pipeline là toàn bộ quá trình CI/CD từ khi trigger đến khi kết thúc. Một Pipeline có thể chứa nhiều Workflow. Workflow điều phối các Job bên trong nó.

**Khi nào dùng `requires`?**
> Khi bạn muốn đảm bảo tính đúng đắn theo thứ tự — ví dụ: chỉ deploy nếu test pass, chỉ build nếu lint sạch.

---

## 📖 Đọc Tiếp

- [1-sequential-workflow.md](1-sequential-workflow.md) — Luồng tuần tự chi tiết
- [2-parallel-workflow.md](2-parallel-workflow.md) — Song song hóa để tăng tốc
- [3-approval-jobs.md](3-approval-jobs.md) — Cổng duyệt thủ công
- [4-scheduled-pipelines.md](4-scheduled-pipelines.md) — Lập lịch tự động
- [5-branch-tag-filters.md](5-branch-tag-filters.md) — Bộ lọc nhánh và tag

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
