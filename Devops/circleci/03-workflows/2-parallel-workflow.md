# Parallel Workflow — Luồng Công Việc Song Song

> **Parallel Workflow** (Luồng Song Song) là kỹ thuật chạy nhiều Job **đồng thời** trong cùng một Workflow để giảm thiểu tổng thời gian build. Hai mẫu thiết kế cốt lõi: **Fan-out** (Phân Tỏa) và **Fan-in** (Tập Hợp).

---

## 📚 Mục Lục

1. [Nguyên Lý Parallel Execution](#1-nguyên-lý-parallel-execution)
2. [Fan-out Pattern — Mẫu Phân Tỏa](#2-fan-out-pattern--mẫu-phân-tỏa)
3. [Fan-in Pattern — Mẫu Tập Hợp](#3-fan-in-pattern--mẫu-tập-hợp)
4. [Fan-out/Fan-in Kết Hợp](#4-fan-outfan-in-kết-hợp)
5. [Parallelism — Song Song Hóa Trong Một Job](#5-parallelism--song-song-hóa-trong-một-job)
6. [Tối Ưu Thời Gian với Parallel](#6-tối-ưu-thời-gian-với-parallel)
7. [Giới Hạn Concurrency — Độ Song Song](#7-giới-hạn-concurrency--độ-song-song)
8. [Mẫu Thực Tế](#8-mẫu-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Nguyên Lý Parallel Execution

Trong CircleCI, bất kỳ Job nào **không có `requires`** hoặc **có cùng điểm xuất phát** sẽ tự động chạy song song.

### Mặc Định: Song Song Khi Không Có `requires`

```yaml
workflows:
  parallel-by-default:
    jobs:
      - lint           # ┐
      - unit-test      # ├── Ba job này khởi động cùng một lúc
      - security-scan  # ┘
```

**Minh họa thời gian:**
```
t=0s  lint          ████████ (8s)
t=0s  unit-test     ████████████████ (16s)
t=0s  security-scan ██████████ (10s)
                 ↑
Total: 16s (bằng job chậm nhất)
# Nếu chạy tuần tự: 8 + 16 + 10 = 34s
```

**Tiết kiệm 53% thời gian** so với sequential!

---

## 2. Fan-out Pattern — Mẫu Phân Tỏa

**Fan-out** (Phân Tỏa): Từ một Job, chia thành **nhiều Job con** chạy song song.

```
Hình dạng: một điểm tỏa ra nhiều nhánh
         ┌── Job B
Job A ───┤── Job C
         └── Job D
```

### Ví Dụ: Build Sau Đó Test Song Song

```yaml
version: 2.1

executors:
  default:
    docker:
      - image: cimg/node:20.0

jobs:
  install:
    executor: default
    steps:
      - checkout
      - run: npm ci
      - persist_to_workspace:
          root: .
          paths: [node_modules/]

  # Bốn job sau đây là "fan-out" từ install
  lint-js:
    executor: default
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run lint:js

  lint-css:
    executor: default
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run lint:css

  unit-test:
    executor: default
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run test:unit

  type-check:
    executor: default
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run type-check

workflows:
  fan-out-example:
    jobs:
      - install                         # Điểm xuất phát
      - lint-js:
          requires: [install]           # ┐
      - lint-css:
          requires: [install]           # ├── Fan-out: 4 job chạy song song
      - unit-test:
          requires: [install]           # │   sau khi install xong
      - type-check:
          requires: [install]           # ┘
```

**Minh họa:**
```
install ██████████████████ (18s)
                          ↓ Fan-out (phân tỏa)
lint-js    ████████ (8s)
lint-css   ██████ (6s)
unit-test  ████████████████ (16s)
type-check █████████ (9s)
Total: 18 + 16 = 34s (thay vì 18 + 8 + 6 + 16 + 9 = 57s)
```

---

## 3. Fan-in Pattern — Mẫu Tập Hợp

**Fan-in** (Tập Hợp): Nhiều Job chờ **tất cả** hoàn thành trước khi một Job tiếp theo bắt đầu.

```
Hình dạng: nhiều nhánh hội tụ về một điểm
Job B ─┐
Job C ─┼── Job E
Job D ─┘
```

### Ví Dụ: Build Chỉ Sau Khi Tất Cả Checks Xanh

```yaml
workflows:
  fan-in-example:
    jobs:
      # Các check chạy song song
      - lint
      - unit-test
      - integration-test
      - security-scan

      # Fan-in: build chờ TẤT CẢ 4 job trên
      - build:
          requires:
            - lint
            - unit-test
            - integration-test
            - security-scan
```

**Hành vi quan trọng của Fan-in:**
- `build` bắt đầu sau khi **job cuối cùng** trong danh sách `requires` hoàn thành
- Nếu **bất kỳ** job nào fail → `build` bị hủy ngay lập tức
- Không có cách để `build` "bỏ qua" một job fail trong `requires`

---

## 4. Fan-out/Fan-in Kết Hợp

Đây là mẫu phổ biến nhất trong production — kết hợp cả hai:

```
install ──fan-out──► lint     ─┐
                   ► unit-test ─┼──fan-in──► build ──► deploy
                   ► security  ─┘
```

### Pipeline Hoàn Chỉnh

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
            - npm-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: npm-{{ checksum "package-lock.json" }}
          paths: [~/.npm]
      - persist_to_workspace:
          root: .
          paths: [node_modules/]

  lint:
    executor: node
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run lint

  unit-test:
    executor: node
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run test:unit
      - store_test_results:
          path: reports/junit/

  integration-test:
    executor: node
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run test:integration

  build:
    executor: node
    steps:
      - checkout
      - attach_workspace: {at: .}
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy:
    executor: node
    steps:
      - attach_workspace: {at: .}
      - run: ./scripts/deploy.sh

workflows:
  fan-out-fan-in:
    jobs:
      # Tầng 1: Cài đặt
      - install

      # Tầng 2: Fan-out — 3 job song song
      - lint:
          requires: [install]
      - unit-test:
          requires: [install]
      - integration-test:
          requires: [install]

      # Tầng 3: Fan-in — build chờ cả 3
      - build:
          requires:
            - lint
            - unit-test
            - integration-test

      # Tầng 4: Deploy
      - deploy:
          requires: [build]
          filters:
            branches:
              only: main
```

**Phân tích thời gian:**
```
install:          ████████████ 12s
                              ↓ fan-out
lint:             ████ 4s
unit-test:        ████████████ 12s     ← bottleneck (nút thắt cổ chai)
integration-test: ████████ 8s
                                     ↓ fan-in (chờ unit-test xong)
build:                               ██████ 6s
deploy:                                    ████ 4s
                                                 ↑
Total: 12 + 12 + 6 + 4 = 34s
So sánh sequential: 12 + 4 + 12 + 8 + 6 + 4 = 46s
```

**Tiết kiệm 26%** — và phần tiết kiệm lớn hơn khi có nhiều test hơn.

---

## 5. Parallelism — Song Song Hóa Trong Một Job

Khác với parallel jobs (nhiều Job song song), `parallelism` — Song Song Hóa chia **một Job** thành nhiều container để chạy cùng lúc, kết hợp với `circleci tests split`.

```yaml
jobs:
  test:
    docker:
      - image: cimg/node:20.0
    parallelism: 4              # Tạo 4 container chạy cùng lúc
    steps:
      - checkout
      - run: npm ci
      - run:
          name: Chia và chạy tests song song
          command: |
            # circleci tests split tự động phân chia danh sách test file
            # --split-by=timings: dựa trên thời gian chạy lịch sử
            TESTS=$(circleci tests glob "src/**/*.test.ts" | \
                    circleci tests split --split-by=timings)
            npx jest $TESTS --ci --reporters=default --reporters=jest-junit
      - store_test_results:
          path: reports/junit/

workflows:
  with-parallelism:
    jobs:
      - test    # 4 container chạy song song, mỗi container chạy 1/4 test suite
```

**Nguyên lý hoạt động:**
```
Không có parallelism:
Container 1: [test1] [test2] [test3] [test4] = 40s

Có parallelism: 4:
Container 1: [test1]          = 10s  ┐
Container 2:         [test2]  = 10s  ├── 4 container song song
Container 3:                  ...    │
Container 4:                  ...    ┘
Total: ~10s (tốc độ tăng 4x)
```

> **Lưu ý:** `parallelism` và **parallel jobs** là hai khái niệm khác nhau:
> - **Parallel Jobs**: nhiều Job khác nhau chạy cùng lúc
> - **`parallelism` key**: một Job được chia thành nhiều instance chạy cùng lúc

---

## 6. Tối Ưu Thời Gian với Parallel

### Nguyên Tắc Nhóm Job

**Nhóm các Job có thể chạy độc lập:**

```yaml
# ✅ TỐT — nhóm theo phụ thuộc thực sự
workflows:
  optimized:
    jobs:
      # Nhóm 1: Không phụ thuộc gì → chạy ngay
      - lint
      - security-scan
      - dependency-audit

      # Nhóm 2: Cần code checkout nhưng không cần nhau
      - unit-test:
          requires: [lint]
      - e2e-test:
          requires: [lint]

      # Nhóm 3: Cần tất cả nhóm 2 xong
      - build:
          requires: [unit-test, e2e-test, security-scan, dependency-audit]

# ❌ KHÔNG TỐT — thêm phụ thuộc không cần thiết
workflows:
  over-sequential:
    jobs:
      - lint
      - security-scan:
          requires: [lint]        # security-scan không phụ thuộc lint!
      - unit-test:
          requires: [security-scan]
      - build:
          requires: [unit-test]
```

### Xác Định Bottleneck — Nút Thắt Cổ Chai

Bottleneck là Job mất nhiều thời gian nhất trong một giai đoạn parallel — nó quyết định tổng thời gian của giai đoạn đó.

```
Cách tìm bottleneck:
1. Vào CircleCI UI → Workflow run
2. Nhìn vào biểu đồ Gantt (thời gian từng job)
3. Job kéo dài nhất trong một nhóm parallel = bottleneck

Cách xử lý bottleneck:
- Nếu là test: dùng parallelism để chia nhỏ
- Nếu là build: tối ưu build steps, dùng caching
- Nếu là lý do khác: xem lại resource_class (CPU/RAM)
```

---

## 7. Giới Hạn Concurrency — Độ Song Song

### Concurrency Limits Của CircleCI

CircleCI giới hạn số Job chạy đồng thời tùy theo plan (gói dịch vụ):

| Plan | Concurrent Jobs | Parallelism Tối Đa |
| ---- | --------------- | ------------------ |
| Free | 1 | 2 |
| Performance | 80+ | 30 |
| Scale | Không giới hạn | Không giới hạn |

Khi vượt giới hạn, Job được đưa vào hàng đợi (queued) cho đến khi có slot trống.

### Kiểm Soát Concurrency Bằng `name`

```yaml
workflows:
  controlled-parallel:
    jobs:
      - test-chrome:
          name: browser-test-chrome    # Đặt tên để phân biệt
      - test-firefox:
          name: browser-test-firefox
      - test-safari:
          name: browser-test-safari
```

### Dùng `matrix` Để Tạo Job Song Song Tự Động

```yaml
jobs:
  test-node-version:
    parameters:
      node-version:
        type: string
    docker:
      - image: cimg/node:<< parameters.node-version >>
    steps:
      - checkout
      - run: npm test

workflows:
  test-multiple-versions:
    jobs:
      - test-node-version:
          matrix:
            parameters:
              node-version: ["18.0", "20.0", "22.0"]
          # Tự động tạo 3 jobs: test-node-version-18.0, ...-20.0, ...-22.0
          # Tất cả chạy song song!
```

---

## 8. Mẫu Thực Tế

### Mẫu A: Multi-Service Pipeline (Microservices)

```yaml
version: 2.1

jobs:
  test-service-a:
    docker: [{image: cimg/python:3.11}]
    steps:
      - checkout
      - run: cd services/service-a && pip install -r requirements.txt && pytest

  test-service-b:
    docker: [{image: cimg/node:20.0}]
    steps:
      - checkout
      - run: cd services/service-b && npm ci && npm test

  test-service-c:
    docker: [{image: cimg/go:1.21}]
    steps:
      - checkout
      - run: cd services/service-c && go test ./...

  build-all:
    docker: [{image: cimg/base:stable}]
    steps:
      - checkout
      - run: docker compose build

  deploy-all:
    docker: [{image: cimg/base:stable}]
    steps:
      - run: ./scripts/deploy-all.sh

workflows:
  microservices-pipeline:
    jobs:
      # Test tất cả services song song
      - test-service-a
      - test-service-b
      - test-service-c

      # Build sau khi tất cả test xanh
      - build-all:
          requires:
            - test-service-a
            - test-service-b
            - test-service-c

      # Deploy sau khi build xong
      - deploy-all:
          requires: [build-all]
          filters:
            branches:
              only: main
```

### Mẫu B: Multi-Platform Build (Cross-Platform)

```yaml
version: 2.1

jobs:
  build-linux:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run: make build-linux
      - persist_to_workspace:
          root: .
          paths: [dist/linux/]

  build-macos:
    macos:
      xcode: "15.0"
    steps:
      - checkout
      - run: make build-macos
      - persist_to_workspace:
          root: .
          paths: [dist/macos/]

  build-windows:
    machine:
      image: windows-server-2022-gui:current
      shell: powershell.exe
    steps:
      - checkout
      - run: make build-windows
      - persist_to_workspace:
          root: .
          paths: [dist/windows/]

  package-release:
    docker:
      - image: cimg/base:stable
    steps:
      - attach_workspace:
          at: .
      - run: |
          ls dist/       # dist/linux/ dist/macos/ dist/windows/ đều có mặt
          ./scripts/create-release.sh

workflows:
  cross-platform-build:
    jobs:
      # Ba platform build song song
      - build-linux
      - build-macos
      - build-windows

      # Đóng gói sau khi cả ba hoàn thành
      - package-release:
          requires:
            - build-linux
            - build-macos
            - build-windows
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Fan-out và Fan-in trong CircleCI là gì?**

> **Fan-out** (Phân Tỏa): một Job tạo ra nhiều Job con chạy song song — ví dụ sau bước `install`, chạy đồng thời `lint`, `test`, và `security-scan`. **Fan-in** (Tập Hợp): nhiều Job song song hội tụ về một điểm — Job tiếp theo chờ tất cả các job trong `requires` xong mới bắt đầu.

**Q: Parallel Jobs và `parallelism` key khác nhau như thế nào?**

> **Parallel Jobs**: nhiều Job *khác nhau* chạy đồng thời trong workflow — mỗi Job làm việc khác nhau (lint, test, build). **`parallelism` key**: chia *một Job* thành nhiều container giống hệt nhau, mỗi container xử lý một phần test suite — thường dùng với `circleci tests split`.

**Q: Làm thế nào để biết bottleneck ở đâu trong parallel workflow?**

> Vào CircleCI UI xem biểu đồ Gantt của Workflow run — Job nào kéo dài nhất trong một giai đoạn parallel là bottleneck. Cách xử lý: nếu là test suite lớn, dùng `parallelism` + `circleci tests split --split-by=timings` để chia nhỏ; nếu là bước build chậm, tăng `resource_class` hoặc cải thiện caching.

**Q: Khi nào dùng Parallel Workflow và khi nào dùng Sequential?**

> **Parallel**: cho các công việc độc lập không cần output của nhau — lint, test, security scan, type check. **Sequential**: khi có thứ tự bắt buộc — build artifact xong mới deploy, test xanh mới build production. Trong thực tế, pipeline tốt thường kết hợp cả hai: fan-out song song để kiểm tra, fan-in tập hợp, rồi sequential để deploy.

---

## 📖 Đọc Tiếp

- [1-sequential-workflow.md](1-sequential-workflow.md) — Tìm hiểu chi tiết `requires` và chuỗi phụ thuộc
- [3-approval-jobs.md](3-approval-jobs.md) — Thêm manual gate vào cuối Fan-in
- [05-optimization/3-test-splitting.md](../05-optimization/3-test-splitting.md) — Chi tiết về `circleci tests split`

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
