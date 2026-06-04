# Parallelism — Song Song Hóa

> Parallelism (song song hóa) trong CircleCI cho phép chạy cùng một job trên nhiều container đồng thời, mỗi container xử lý một phần công việc. Đây là công cụ mạnh nhất để giảm thời gian build khi test suite lớn.

---

## 🧠 Khái Niệm Cốt Lõi

### Parallelism Hoạt Động Như Thế Nào?

```yaml
jobs:
  test:
    parallelism: 4   # Tạo 4 container chạy cùng lúc
    steps:
      - run: echo "Container $CIRCLE_NODE_INDEX / $CIRCLE_NODE_TOTAL"
```

Kết quả: CircleCI khởi động **4 container giống hệt nhau** và chạy song song:

```
Container 0: echo "Container 0 / 4"
Container 1: echo "Container 1 / 4"
Container 2: echo "Container 2 / 4"
Container 3: echo "Container 3 / 4"
```

> Nếu không có test splitting, tất cả 4 container sẽ chạy **toàn bộ** test — không có lợi gì! Cần kết hợp với `circleci tests split` để mỗi container chạy **một phần**.

---

## ⚙️ Cú Pháp

### Cấu Hình Cơ Bản

```yaml
jobs:
  test:
    docker:
      - image: cimg/node:20.0
    parallelism: 4       # Số container song song
    resource_class: medium
    steps:
      - checkout
      - run:
          name: Chạy kiểm thử song song
          command: |
            circleci tests glob "src/**/*.test.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest
```

| Giá Trị `parallelism` | Ý Nghĩa |
| --------------------- | ------- |
| `1` (mặc định) | Chạy tuần tự, 1 container |
| `2` | 2 container, ~50% thời gian |
| `4` | 4 container, ~25% thời gian |
| `8` | 8 container, ~12.5% thời gian |
| `20` | 20 container (giới hạn tùy plan) |

---

## 🔢 Biến Môi Trường Quan Trọng

| Biến | Mô Tả | Ví Dụ |
| ---- | ----- | ----- |
| `CIRCLE_NODE_INDEX` | Index 0-based của container hiện tại | `0`, `1`, `2`, `3` |
| `CIRCLE_NODE_TOTAL` | Tổng số container (= parallelism) | `4` |

```bash
# Kiểm tra trong log:
echo "Tôi là container $CIRCLE_NODE_INDEX trong tổng số $CIRCLE_NODE_TOTAL"
# → "Tôi là container 2 trong tổng số 4"
```

---

## 📊 Parallelism + Test Splitting — Kết Hợp Hoàn Hảo

### Luồng Đầy Đủ

```
circleci tests glob "src/**/*.test.ts"
  → [test1.ts, test2.ts, ..., test100.ts]  (100 files)

circleci tests split --split-by=timings
  Container 0: [test1.ts, test5.ts, test9.ts, ...]    ← ~25 files, ~5 phút
  Container 1: [test2.ts, test6.ts, test10.ts, ...]   ← ~25 files, ~5 phút
  Container 2: [test3.ts, test7.ts, test11.ts, ...]   ← ~25 files, ~5 phút
  Container 3: [test4.ts, test8.ts, test12.ts, ...]   ← ~25 files, ~5 phút

Tổng thời gian: max(5, 5, 5, 5) = 5 phút  (thay vì 20 phút)
```

### Ví Dụ Hoàn Chỉnh — Node.js

```yaml
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/node:20.0
    parallelism: 4
    resource_class: medium
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - node-v1-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: node-v1-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
      
      - run:
          name: Kiểm thử song song với phân chia theo thời gian
          command: |
            mkdir -p test-results
            
            circleci tests glob "src/**/*.test.ts" "src/**/*.spec.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest \
              --forceExit \
              --reporters=default \
              --reporters=jest-junit
          environment:
            JEST_JUNIT_OUTPUT_DIR: test-results
      
      # Quan trọng: lưu kết quả để timing cải thiện theo thời gian
      - store_test_results:
          path: test-results
      
      - store_artifacts:
          path: test-results

workflows:
  ci:
    jobs:
      - test
```

---

## 🏗️ Kết Hợp Parallelism Với Workflow

### Fan-Out Pattern — Mô Hình Phân Tỏa

Dùng parallelism kết hợp với workflow song song để tối ưu tối đa:

```yaml
jobs:
  install:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - persist_to_workspace:
          root: .
          paths:
            - node_modules/

  test-unit:
    docker:
      - image: cimg/node:20.0
    parallelism: 4      # 4 container cho unit tests
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: |
          circleci tests glob "src/**/*.unit.test.ts" | \
          circleci tests split --split-by=timings | \
          xargs npx jest
      - store_test_results:
          path: test-results

  test-integration:
    docker:
      - image: cimg/node:20.0
      - image: postgres:15    # Database service container
        environment:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
    parallelism: 2      # Ít hơn vì integration tests tốn resource hơn
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: |
          circleci tests glob "src/**/*.integration.test.ts" | \
          circleci tests split --split-by=timings | \
          xargs npx jest
      - store_test_results:
          path: test-results

  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/

workflows:
  ci-cd:
    jobs:
      - install
      
      # Fan-out — Phân Tỏa: test-unit, test-integration, build chạy song song
      - test-unit:
          requires: [install]
      - test-integration:
          requires: [install]
      - build:
          requires: [install]
      
      # Fan-in — Tập Hợp: chỉ deploy khi tất cả xanh
      - deploy:
          requires:
            - test-unit
            - test-integration
            - build
```

---

## 💰 Tính Toán Chi Phí — Credit Calculation

CircleCI tính credit theo công thức:

```
Total Credits = parallelism × resource_class_rate × minutes

resource_class_rate (credits/phút):
  small (1 vCPU, 2GB):     5 credits/phút
  medium (2 vCPU, 4GB):   10 credits/phút
  large (4 vCPU, 8GB):    20 credits/phút
  xlarge (8 vCPU, 16GB):  40 credits/phút
```

### So Sánh Hiệu Quả Chi Phí

| Cấu Hình | Thời Gian | Credits/Phút | Tổng Credits | Tốc Độ |
| -------- | --------- | ------------ | ------------ | ------ |
| medium × 1 | 20 phút | 10 | **200** | Chuẩn |
| medium × 2 | 11 phút | 20 | **220** | +9% chi phí, 45% nhanh hơn |
| medium × 4 | 6 phút | 40 | **240** | +20% chi phí, 70% nhanh hơn |
| large × 2 | 8 phút | 40 | **320** | +60% chi phí, 60% nhanh hơn |
| small × 8 | 5 phút | 40 | **200** | Cùng chi phí, 75% nhanh hơn ⭐ |

> **Mẹo tối ưu chi phí:** `small × 8` thường hiệu quả hơn `medium × 4` — cùng tổng credits nhưng nhanh hơn.

---

## 🔬 Resource Class — Lớp Tài Nguyên

### Docker Executor Resource Classes

| Resource Class | vCPU | RAM | Credits/Phút | Phù Hợp |
| -------------- | ---- | --- | ------------ | -------- |
| `small` | 1 | 2 GB | 5 | Script đơn giản, lint |
| `medium` (mặc định) | 2 | 4 GB | 10 | Node.js, Python, Go thông thường |
| `medium+` | 3 | 6 GB | 15 | Ứng dụng cần nhiều RAM hơn |
| `large` | 4 | 8 GB | 20 | Build nặng, nhiều worker |
| `xlarge` | 8 | 16 GB | 40 | Java enterprise, Scala |
| `2xlarge` | 16 | 32 GB | 80 | Hiếm gặp, rất tốn kém |

### Machine Executor Resource Classes (Linux VM)

| Resource Class | vCPU | RAM | Credits/Phút |
| -------------- | ---- | --- | ------------ |
| `medium` | 2 | 7.5 GB | 10 |
| `large` | 4 | 15 GB | 20 |
| `xlarge` | 8 | 32 GB | 40 |
| `2xlarge` | 16 | 64 GB | 80 |

### Cách Chọn Resource Class Phù Hợp

```yaml
# Chọn dựa trên nhu cầu thực tế
jobs:
  lint:
    resource_class: small       # Lint chỉ cần 1 CPU
  
  test:
    resource_class: medium      # Test thông thường
    parallelism: 4
  
  build-docker:
    machine:
      image: ubuntu-2204:current
    resource_class: large       # Docker build cần nhiều CPU
  
  android-build:
    macos:
      xcode: "15.0"
    resource_class: macos.m1.medium.gen1  # macOS ARM
```

---

## 📏 Giới Hạn Parallelism

| Plan | Parallelism Tối Đa |
| ---- | ----------------- |
| Free | 4 |
| Performance | 80 |
| Scale | Tùy chỉnh |
| Server (self-hosted) | Không giới hạn |

---

## 🧪 Parallelism Mà Không Cần Test Splitting — Edge Cases

Có những trường hợp dùng parallelism mà không cần `circleci tests split`:

### Trường Hợp 1: Chạy Nhiều Test Loại Khác Nhau

```yaml
jobs:
  test-matrix:
    parallelism: 3
    steps:
      - run:
          command: |
            case $CIRCLE_NODE_INDEX in
              0) npm run test:unit ;;
              1) npm run test:integration ;;
              2) npm run test:e2e ;;
            esac
```

### Trường Hợp 2: Multi-Database Testing

```yaml
jobs:
  test-databases:
    parallelism: 3
    steps:
      - run:
          command: |
            DATABASES=("postgres" "mysql" "sqlite")
            DB=${DATABASES[$CIRCLE_NODE_INDEX]}
            DATABASE_URL="$DB://..." npm test
```

### Trường Hợp 3: Multi-Region Deployment

```yaml
jobs:
  deploy-regions:
    parallelism: 3
    steps:
      - run:
          command: |
            REGIONS=("us-east-1" "eu-west-1" "ap-southeast-1")
            REGION=${REGIONS[$CIRCLE_NODE_INDEX]}
            aws s3 sync dist/ s3://my-app-$REGION --region $REGION
```

---

## ⚠️ Các Lỗi Thường Gặp

### Lỗi 1: Dùng Parallelism Mà Không Split

```yaml
# ❌ Mỗi container chạy toàn bộ test — lãng phí!
jobs:
  test:
    parallelism: 4
    steps:
      - run: npx jest   # Tất cả 4 container đều chạy 100% test

# ✅ Cần kết hợp với circleci tests split
jobs:
  test:
    parallelism: 4
    steps:
      - run: |
          circleci tests glob "**/*.test.ts" | \
          circleci tests split --split-by=timings | \
          xargs npx jest
```

### Lỗi 2: Race Condition Khi Ghi File Kết Quả

```yaml
# ❌ Tất cả container ghi cùng file → corrupt
- store_test_results:
    path: test-results/junit.xml   # Conflict!

# ✅ Dùng NODE_INDEX để tên file unique
- run: npx jest --outputFile=test-results/junit-${CIRCLE_NODE_INDEX}.xml
- store_test_results:
    path: test-results/
```

### Lỗi 3: Shared State Giữa Các Container

```bash
# ❌ Database seed chạy ở tất cả container → conflict
npx sequelize db:seed:all   # 4 container đều seed → duplicate data

# ✅ Chỉ seed ở container 0
if [ "$CIRCLE_NODE_INDEX" -eq 0 ]; then
  npx sequelize db:seed:all
fi
sleep 5   # Chờ seed xong
npm test
```

---

## 🎯 Quy Trình Tối Ưu Hóa Từng Bước

```
Bước 1: Đo thời gian hiện tại
  → Dùng Pipeline Insights xem job nào chậm nhất

Bước 2: Thêm parallelism: 2 và test splitting
  → Chạy thử, đo lại

Bước 3: Chờ 5–10 lần chạy để timing data tích lũy
  → Splitting sẽ chính xác hơn

Bước 4: Tăng parallelism nếu thấy còn chậm
  → Thường parallelism: 4 là điểm tối ưu cost/time

Bước 5: Điều chỉnh resource_class
  → Thử small × 8 thay vì medium × 4

Bước 6: Theo dõi Pipeline Insights định kỳ
  → Phát hiện regression và cơ hội tối ưu mới
```

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Parallelism trong CircleCI là gì?**

A: Parallelism (song song hóa) là tính năng cho phép chạy nhiều instance của cùng một job đồng thời — mỗi container nhận một phần công việc thông qua `CIRCLE_NODE_INDEX`. Kết hợp với `circleci tests split`, mỗi container chỉ chạy một phần test suite, giúp giảm tổng thời gian build.

**Q: Nếu tôi đặt parallelism: 4 mà không dùng test splitting, chuyện gì xảy ra?**

A: Cả 4 container sẽ chạy **toàn bộ** test suite, tốn gấp 4 lần credits mà không tiết kiệm được thời gian. Parallelism chỉ có ý nghĩa khi kết hợp với test splitting hoặc tự chia công việc bằng `CIRCLE_NODE_INDEX`.

**Q: Làm thế nào để chọn số parallelism tối ưu?**

A: Bắt đầu với parallelism bằng `ceil(thời_gian_hiện_tại / thời_gian_mục_tiêu)`. Ví dụ: test chạy 20 phút, mục tiêu < 5 phút → parallelism: 4. Sau đó điều chỉnh dựa trên chi phí thực tế và Pipeline Insights.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
