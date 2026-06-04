# Sequential Workflow — Luồng Công Việc Tuần Tự

> **Sequential Workflow** (Luồng Tuần Tự) là mẫu workflow trong đó các Job chạy **lần lượt theo thứ tự**, mỗi Job chỉ bắt đầu sau khi Job trước đó hoàn thành thành công. Từ khóa cốt lõi: `requires`.

---

## 📚 Mục Lục

1. [Khái Niệm `requires`](#1-khái-niệm-requires)
2. [Chuỗi Phụ Thuộc Đơn Giản](#2-chuỗi-phụ-thuộc-đơn-giản)
3. [Phụ Thuộc Nhiều Job](#3-phụ-thuộc-nhiều-job)
4. [Fail-Fast — Thất Bại Nhanh](#4-fail-fast--thất-bại-nhanh)
5. [Truyền Dữ Liệu Giữa Các Job](#5-truyền-dữ-liệu-giữa-các-job)
6. [Mẫu Sequential Thực Tế](#6-mẫu-sequential-thực-tế)
7. [So Sánh Sequential vs Parallel](#7-so-sánh-sequential-vs-parallel)
8. [Các Lỗi Thường Gặp](#8-các-lỗi-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm `requires`

`requires` là khóa cấu hình trong section `workflows` xác định một Job **phụ thuộc** vào Job khác. Job sẽ không bắt đầu cho đến khi **tất cả** các Job trong danh sách `requires` hoàn thành với trạng thái **success** (thành công).

### Cú Pháp Cơ Bản

```yaml
workflows:
  my-workflow:
    jobs:
      - job-a                   # Không requires → chạy ngay
      - job-b:
          requires:
            - job-a             # job-b chờ job-a thành công
      - job-c:
          requires:
            - job-b             # job-c chờ job-b thành công
```

**Kết quả:** `job-a` → `job-b` → `job-c` (tuần tự)

### Các Trạng Thái Job

| Trạng Thái | Mô Tả | Ảnh Hưởng đến Job Phụ Thuộc |
| ---------- | ------ | --------------------------- |
| `success` | Job hoàn thành thành công | Job phụ thuộc được khởi chạy |
| `failed` | Job thất bại (exit code ≠ 0) | Job phụ thuộc bị hủy (cancelled) |
| `cancelled` | Job bị hủy thủ công | Job phụ thuộc bị hủy |
| `on_hold` | Đang chờ approval | Job phụ thuộc chờ tiếp |
| `running` | Đang thực thi | Job phụ thuộc chờ |

---

## 2. Chuỗi Phụ Thuộc Đơn Giản

Mẫu phổ biến nhất: `A → B → C → D`

```yaml
version: 2.1

jobs:
  install-dependencies:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run:
          name: Cài đặt dependencies
          command: npm ci
      - persist_to_workspace:
          root: .
          paths: [node_modules/]

  lint:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Kiểm tra linting
          command: npm run lint

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Chạy unit tests
          command: npm test

  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Build ứng dụng
          command: npm run build

workflows:
  sequential-ci:
    jobs:
      - install-dependencies                    # Bước 1: Luôn chạy đầu tiên
      - lint:
          requires:
            - install-dependencies              # Bước 2: Sau khi cài xong
      - test:
          requires:
            - lint                              # Bước 3: Sau khi lint sạch
      - build:
          requires:
            - test                              # Bước 4: Sau khi test xanh
```

**Minh họa thời gian:**
```
t=0s    install-dependencies ████████████ (30s)
t=30s                                    lint ████ (10s)
t=40s                                         test ████████ (20s)
t=60s                                                  build ██████ (15s)
Total: 75s (tuần tự)
```

---

## 3. Phụ Thuộc Nhiều Job

Một Job có thể `requires` nhiều Job — nó chờ **tất cả** hoàn thành:

```yaml
workflows:
  multi-gate:
    jobs:
      - lint
      - unit-test
      - security-scan

      # build phải chờ CẢ BA job trên xong mới chạy
      - build:
          requires:
            - lint
            - unit-test
            - security-scan

      - deploy:
          requires:
            - build
```

**Minh họa:**
```
lint          ████████
unit-test     ████████████████
security-scan ██████████
                              build ███████
                                          deploy ████
```

`build` bắt đầu sau khi job chậm nhất (`unit-test`) xong — đây là **Fan-in Pattern** (Mẫu Tập Hợp).

### Cây Phụ Thuộc Phức Tạp

```yaml
workflows:
  complex-sequential:
    jobs:
      # Tầng 1: Song song
      - lint
      - type-check

      # Tầng 2: Chờ tầng 1
      - unit-test:
          requires: [lint, type-check]

      # Tầng 3: Chờ tầng 2
      - integration-test:
          requires: [unit-test]

      # Tầng 4: Chờ tầng 3
      - e2e-test:               # E2E — End-to-End Test — Kiểm Thử Đầu Cuối
          requires: [integration-test]

      # Tầng 5: Chờ tầng 4
      - build:
          requires: [e2e-test]
```

---

## 4. Fail-Fast — Thất Bại Nhanh

Khi một Job trong chuỗi thất bại, **tất cả Job phụ thuộc bị hủy ngay lập tức** — đây gọi là cơ chế **fail-fast**.

```
Ví dụ: unit-test fail

lint      ✅ (success)
unit-test ❌ (failed — exit code 1)
build     ⊘ (cancelled — không bao giờ chạy)
deploy    ⊘ (cancelled — không bao giờ chạy)
```

**Lợi ích của fail-fast:**
- Tiết kiệm credits — không chạy những bước vô nghĩa sau khi đã fail
- Phản hồi nhanh hơn cho developer — biết lỗi sớm, sửa sớm
- Không deploy code lỗi lên bất kỳ môi trường nào

### Xử Lý Khi Job Fail

```yaml
jobs:
  test-with-notification:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm test
      - run:
          name: Thông báo khi fail
          command: ./notify-team.sh "Test thất bại!"
          when: on_fail           # Chỉ chạy step này khi job fail
```

**`when` trong step:**
- `always` — luôn chạy dù job pass hay fail (dùng cho cleanup, notification)
- `on_success` — chỉ khi thành công (mặc định)
- `on_fail` — chỉ khi thất bại (dùng để debug, thông báo)

---

## 5. Truyền Dữ Liệu Giữa Các Job

Vì mỗi Job chạy trong container riêng biệt, cần cơ chế để truyền file giữa các Job.

### Cách 1: Workspace — Không Gian Làm Việc Chung

```yaml
jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci && npm run build
      - persist_to_workspace:     # Lưu file vào workspace tạm thời
          root: .
          paths:
            - dist/               # Chỉ lưu thư mục dist/
            - node_modules/

  deploy:
    docker:
      - image: cimg/node:20.0
    steps:
      - attach_workspace:         # Lấy file từ workspace
          at: .                   # Gắn vào thư mục hiện tại
      - run: ls dist/             # dist/ đã có sẵn
      - run: ./deploy.sh

workflows:
  pipeline:
    jobs:
      - build
      - deploy:
          requires: [build]
```

> **Workspace vs Cache:**
> - **Workspace**: truyền file giữa các Job trong **cùng một Workflow run** — dữ liệu tạm thời, không tái sử dụng giữa các lần chạy.
> - **Cache**: tái sử dụng giữa các Pipeline run — dành cho `node_modules`, build artifacts ổn định.

### Cách 2: Artifacts — Tài Nguyên Lưu Trữ

```yaml
jobs:
  test:
    steps:
      - run: npm test -- --coverage
      - store_artifacts:          # Lưu file để download từ UI
          path: coverage/
          destination: test-coverage
      - store_test_results:       # Lưu kết quả test để CircleCI phân tích
          path: test-results/
```

---

## 6. Mẫu Sequential Thực Tế

### Mẫu A: CI/CD Pipeline Node.js Đơn Giản

```yaml
version: 2.1

executors:
  node:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

jobs:
  install:
    executor: node
    steps:
      - checkout
      - restore_cache:
          keys:
            - npm-v1-{{ checksum "package-lock.json" }}
            - npm-v1-
      - run: npm ci
      - save_cache:
          key: npm-v1-{{ checksum "package-lock.json" }}
          paths: [~/.npm]
      - persist_to_workspace:
          root: .
          paths: [node_modules/]

  lint:
    executor: node
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run lint

  test:
    executor: node
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm test -- --ci --coverage
      - store_test_results:
          path: test-results/
      - store_artifacts:
          path: coverage/

  build:
    executor: node
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy-staging:
    executor: node
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy lên môi trường staging
          command: |
            aws s3 sync dist/ s3://my-app-staging --delete
            aws cloudfront create-invalidation \
              --distribution-id $STAGING_CF_ID \
              --paths "/*"

  deploy-production:
    executor: node
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy lên môi trường production
          command: |
            aws s3 sync dist/ s3://my-app-production --delete
            aws cloudfront create-invalidation \
              --distribution-id $PROD_CF_ID \
              --paths "/*"

workflows:
  ci-cd:
    jobs:
      # Giai đoạn 1: Cài đặt
      - install

      # Giai đoạn 2: Kiểm tra chất lượng (song song sau install)
      - lint:
          requires: [install]
      - test:
          requires: [install]

      # Giai đoạn 3: Build (sau khi cả lint và test xanh)
      - build:
          requires: [lint, test]

      # Giai đoạn 4: Deploy staging (chỉ từ main)
      - deploy-staging:
          requires: [build]
          filters:
            branches:
              only: main

      # Giai đoạn 5: Duyệt để lên production
      - approve-production:
          type: approval
          requires: [deploy-staging]
          filters:
            branches:
              only: main

      # Giai đoạn 6: Deploy production
      - deploy-production:
          requires: [approve-production]
          context: aws-production
          filters:
            branches:
              only: main
```

### Mẫu B: Pipeline Docker Build và Push

```yaml
version: 2.1

jobs:
  build-image:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run:
          name: Build Docker image
          command: |
            docker build \
              --tag my-app:$CIRCLE_SHA1 \
              --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
              .

  scan-image:
    machine:
      image: ubuntu-2204:current
    steps:
      - run:
          name: Quét lỗ hổng bảo mật với Trivy
          command: |
            trivy image --exit-code 1 --severity HIGH,CRITICAL my-app:$CIRCLE_SHA1

  push-image:
    machine:
      image: ubuntu-2204:current
    steps:
      - run:
          name: Đăng nhập và push lên ECR — Elastic Container Registry
          command: |
            aws ecr get-login-password | \
              docker login --username AWS \
              --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
            docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:$CIRCLE_SHA1

  deploy-k8s:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Cập nhật Kubernetes deployment
          command: |
            kubectl set image deployment/my-app \
              app=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:$CIRCLE_SHA1

workflows:
  docker-pipeline:
    jobs:
      - build-image
      - scan-image:
          requires: [build-image]           # Quét sau khi build xong
      - push-image:
          requires: [scan-image]            # Chỉ push nếu scan sạch
          context: aws-ecr-credentials
      - deploy-k8s:
          requires: [push-image]            # Deploy sau khi image đã lên registry
          context: kubernetes-production
          filters:
            branches:
              only: main
```

---

## 7. So Sánh Sequential vs Parallel

| Tiêu Chí | Sequential | Parallel |
| -------- | ---------- | -------- |
| **Thứ tự** | Bắt buộc theo thứ tự | Chạy cùng lúc |
| **Thời gian** | Tổng = tổng thời gian từng job | Tổng ≈ job chậm nhất |
| **Credits** | Ít hơn (job sau mới dùng resource) | Nhiều hơn (nhiều container cùng lúc) |
| **Đảm bảo** | Chắc chắn thứ tự thực thi | Không đảm bảo thứ tự |
| **Dùng khi** | Deploy, approval flow | Test, lint, scan |

**Khi nào dùng Sequential?**
- Phải đảm bảo thứ tự logic (build trước deploy sau)
- Job sau phụ thuộc artifact — tài nguyên sinh ra — của job trước
- Cần approval gate giữa các giai đoạn
- Giới hạn resource (ví dụ: môi trường staging chỉ chịu được một deploy lúc)

---

## 8. Các Lỗi Thường Gặp

### Lỗi 1: Circular Dependency — Phụ Thuộc Vòng Tròn

```yaml
# ❌ SAI — job-a và job-b phụ thuộc lẫn nhau
workflows:
  bad:
    jobs:
      - job-a:
          requires: [job-b]
      - job-b:
          requires: [job-a]
```

CircleCI sẽ báo lỗi `Error: Circular dependency detected`. Không có Job nào có thể bắt đầu.

### Lỗi 2: Requires Job Không Tồn Tại

```yaml
# ❌ SAI — tên job viết sai
workflows:
  bad:
    jobs:
      - build
      - deploy:
          requires: [buld]    # Typo! "buld" không tồn tại
```

Lỗi: `Error: Job "deploy" requires job "buld" which does not exist`.

### Lỗi 3: Quên persist_to_workspace

```yaml
# ❌ SAI — build chưa persist nhưng deploy dùng dist/
jobs:
  build:
    steps:
      - run: npm run build   # dist/ tạo ra nhưng không persist

  deploy:
    steps:
      - run: ls dist/        # ❌ dist/ không có ở đây

# ✅ ĐÚNG — thêm persist_to_workspace
jobs:
  build:
    steps:
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy:
    steps:
      - attach_workspace:
          at: .
      - run: ls dist/        # ✅ Có sẵn rồi
```

### Lỗi 4: Thiếu Context Cho Job Cuối Chuỗi

```yaml
# ❌ SAI — deploy cần AWS credentials nhưng không có context
workflows:
  pipeline:
    jobs:
      - build
      - deploy:
          requires: [build]
          # Thiếu: context: aws-production

# ✅ ĐÚNG
workflows:
  pipeline:
    jobs:
      - build
      - deploy:
          requires: [build]
          context: aws-production    # Cung cấp AWS_ACCESS_KEY_ID, etc.
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: `requires` trong CircleCI workflow làm gì?**

> `requires` định nghĩa thứ tự thực thi các Job. Một Job có `requires: [A, B]` chỉ được phép chạy sau khi cả Job A và Job B đều hoàn thành thành công. Nếu bất kỳ job nào trong `requires` thất bại, job hiện tại bị hủy (cancelled) ngay lập tức.

**Q: Tại sao cần Sequential Workflow thay vì để tất cả Job chạy song song?**

> Vì nhiều bước có thứ tự logic bắt buộc: không thể deploy trước khi build xong, không thể push Docker image trước khi quét bảo mật xong. Sequential Workflow đảm bảo những ràng buộc này và ngăn deploy code lỗi.

**Q: Làm thế nào để truyền file giữa các Job trong chuỗi Sequential?**

> Dùng `persist_to_workspace` để lưu file vào Workspace — Không Gian Làm Việc Chung sau mỗi Job, và `attach_workspace` để lấy file đó trong Job tiếp theo. Workspace tồn tại trong suốt một Workflow run và bị xóa khi Workflow kết thúc.

**Q: Fail-fast trong CircleCI là gì? Lợi ích?**

> Khi một Job trong chuỗi `requires` thất bại, tất cả Job phụ thuộc vào nó bị hủy ngay lập tức — không phải chờ timeout. Lợi ích: tiết kiệm credits không chạy bước vô nghĩa, phản hồi nhanh hơn cho developer, và ngăn deploy code chứa lỗi.

---

## 📖 Đọc Tiếp

- [2-parallel-workflow.md](2-parallel-workflow.md) — Kết hợp sequential với parallel để tối ưu tốc độ
- [3-approval-jobs.md](3-approval-jobs.md) — Thêm approval gate vào chuỗi sequential
- [5-branch-tag-filters.md](5-branch-tag-filters.md) — Chỉ chạy một số Job trên nhánh nhất định

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
