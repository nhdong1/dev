# Test Splitting — Phân Chia Kiểm Thử

> Test splitting (phân chia kiểm thử) là kỹ thuật chia nhỏ bộ test suite (tập kiểm thử) thành nhiều phần và chạy đồng thời trên nhiều container, giúp giảm thời gian chờ kết quả kiểm thử từ nhiều chục phút xuống còn vài phút.

---

## 🧠 Khái Niệm Cốt Lõi

### Tại Sao Cần Test Splitting?

```
Không có splitting:
  1 container chạy 1000 test files × 3 giây/test = 50 phút

Với splitting (parallelism: 10):
  10 container × 100 test files mỗi container × 3 giây = 5 phút
  → Tiết kiệm 45 phút (90%)
```

### Nguyên Lý Hoạt Động

```
CIRCLE_NODE_TOTAL = 4  (tổng số container)
CIRCLE_NODE_INDEX = 0, 1, 2, 3  (index của container hiện tại)

Container 0: chạy test files 1–250
Container 1: chạy test files 251–500
Container 2: chạy test files 501–750
Container 3: chạy test files 751–1000
```

---

## ⚙️ Công Cụ `circleci tests split`

### Cú Pháp Cơ Bản

```bash
# Dạng pipeline cơ bản:
<danh-sách-tests> | circleci tests split [--options]

# Ví dụ:
circleci tests glob "**/*.test.js" | circleci tests split
```

### Các Lệnh Con

| Lệnh | Mô Tả |
| ---- | ----- |
| `circleci tests glob` | Tìm file test theo pattern — Mẫu Tìm Kiếm |
| `circleci tests split` | Chia danh sách test cho container hiện tại |
| `circleci tests run` | Chạy test với splitting tích hợp (CircleCI CLI v0.1.20299+) |

---

## 📂 Ba Phương Pháp Splitting

### 1. File-Based Splitting — Phân Chia Theo File (Mặc Định)

Chia đều số lượng file cho các container:

```yaml
- run:
    name: Chạy test phân tán theo file
    command: |
      # Tìm tất cả file test
      circleci tests glob "src/**/*.spec.ts" > /tmp/all-tests.txt
      
      # Chia file cho container hiện tại
      cat /tmp/all-tests.txt | circleci tests split > /tmp/my-tests.txt
      
      # Chạy phần được phân
      cat /tmp/my-tests.txt | xargs npx jest --testPathPattern
```

**Ưu điểm:** Đơn giản, không cần dữ liệu lịch sử
**Nhược điểm:** Không đều nếu các test có thời gian chênh lệch lớn

### 2. Timing-Based Splitting — Phân Chia Theo Thời Gian (Tốt Nhất)

Chia dựa trên thời gian chạy lịch sử của mỗi test file:

```yaml
steps:
  - run:
      name: Chạy test phân tán theo thời gian
      command: |
        circleci tests glob "src/**/*.spec.ts" | \
        circleci tests split --split-by=timings | \
        xargs npx jest --forceExit --testPathPattern
  
  # Lưu kết quả để CircleCI học thời gian cho lần sau
  - store_test_results:
      path: test-results/
```

> **Quan trọng:** Cần `store_test_results` để CircleCI lưu thời gian chạy của từng test — lần đầu sẽ fallback về file-based, từ lần thứ hai trở đi sẽ dùng timing.

**Ưu điểm:** Phân chia đều nhất, container xong gần đồng thời
**Nhược điểm:** Cần lịch sử timing (vài lần chạy đầu sẽ kém chính xác)

### 3. Name-Based Splitting — Phân Chia Theo Tên

Chia dựa trên tên test (hash tên vào bucket):

```yaml
- run:
    command: |
      circleci tests glob "**/*.test.py" | \
      circleci tests split --split-by=name | \
      xargs pytest
```

**Ưu điểm:** Nhất quán — cùng tên luôn vào cùng container
**Nhược điểm:** Không đảm bảo đều về thời gian

---

## 🏗️ Cấu Hình Đầy Đủ

### Node.js — Jest

```yaml
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/node:20.0
    parallelism: 4          # 4 container chạy song song
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
          name: Chạy Jest với phân chia theo thời gian
          command: |
            mkdir -p test-results
            
            # Lấy danh sách file test và phân chia
            TESTS=$(circleci tests glob "src/**/*.test.{js,ts}" | \
                    circleci tests split --split-by=timings)
            
            # Chạy phần được phân
            npx jest \
              --testPathPattern="$(echo $TESTS | tr ' ' '|')" \
              --reporters=default \
              --reporters=jest-junit \
              --forceExit
          environment:
            JEST_JUNIT_OUTPUT_DIR: test-results
            JEST_JUNIT_CLASSNAME: "{classname}"
      
      # Lưu kết quả test để dùng timing lần sau
      - store_test_results:
          path: test-results
      
      # Lưu report để xem trong CircleCI UI
      - store_artifacts:
          path: test-results

workflows:
  ci:
    jobs:
      - test
```

### Python — pytest

```yaml
jobs:
  test:
    docker:
      - image: cimg/python:3.11
    parallelism: 4
    steps:
      - checkout
      - restore_cache:
          keys:
            - pip-v1-{{ checksum "requirements.txt" }}
      - run: pip install -r requirements.txt pytest-split pytest-junit
      - save_cache:
          key: pip-v1-{{ checksum "requirements.txt" }}
          paths:
            - ~/.cache/pip
      
      - run:
          name: Chạy pytest với phân chia
          command: |
            # Lấy danh sách test và phân chia
            circleci tests glob "tests/**/test_*.py" | \
            circleci tests split --split-by=timings | \
            xargs python -m pytest \
              --junitxml=test-results/pytest.xml \
              -v
      
      - store_test_results:
          path: test-results
```

### Ruby — RSpec

```yaml
jobs:
  test:
    docker:
      - image: cimg/ruby:3.2
    parallelism: 3
    steps:
      - checkout
      - restore_cache:
          keys:
            - gems-v1-{{ checksum "Gemfile.lock" }}
      - run: bundle install --path vendor/bundle
      - save_cache:
          key: gems-v1-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle
      
      - run:
          name: Chạy RSpec với phân chia
          command: |
            # Lấy danh sách spec files
            TESTS=$(circleci tests glob "spec/**/*_spec.rb" | \
                    circleci tests split --split-by=timings)
            
            bundle exec rspec \
              --format progress \
              --format RspecJunitFormatter \
              --out test-results/rspec.xml \
              $TESTS
      
      - store_test_results:
          path: test-results
```

### Go — go test

```yaml
jobs:
  test:
    docker:
      - image: cimg/go:1.21
    parallelism: 4
    steps:
      - checkout
      - restore_cache:
          keys:
            - go-mod-v1-{{ checksum "go.sum" }}
      - run: go mod download
      - save_cache:
          key: go-mod-v1-{{ checksum "go.sum" }}
          paths:
            - /home/circleci/go/pkg/mod
      
      - run:
          name: Chạy Go test với phân chia
          command: |
            # Lấy danh sách test packages
            go list ./... | \
            circleci tests split --split-by=timings | \
            xargs go test -v 2>&1 | go-junit-report > test-results/go-test.xml
      
      - store_test_results:
          path: test-results
```

---

## 🔬 `store_test_results` — Lưu Kết Quả Kiểm Thử

Đây là bước **bắt buộc** để timing-based splitting hoạt động tốt:

```yaml
- store_test_results:
    path: test-results/   # Thư mục chứa file XML kết quả (JUnit format)
```

### CircleCI Đọc Gì Từ JUnit XML?

```xml
<!-- Ví dụ JUnit XML output -->
<testsuite name="AuthService" time="15.234">
  <testcase name="should login successfully" time="2.1" />
  <testcase name="should reject invalid token" time="0.8" />
  <testcase name="should refresh token" time="3.2" />
</testsuite>
```

CircleCI lưu `time` của từng test file → dùng để phân chia thông minh hơn.

---

## 📊 Biến Môi Trường Liên Quan

| Biến | Mô Tả | Ví Dụ |
| ---- | ----- | ----- |
| `CIRCLE_NODE_TOTAL` | Tổng số container (parallelism) | `4` |
| `CIRCLE_NODE_INDEX` | Index của container hiện tại (bắt đầu từ 0) | `0`, `1`, `2`, `3` |

### Sử Dụng Trực Tiếp Trong Script

```bash
# Thay vì dùng circleci tests split, tự chia thủ công:
ALL_TESTS=(test1.py test2.py test3.py test4.py test5.py test6.py test7.py test8.py)
MY_TESTS=()

for i in "${!ALL_TESTS[@]}"; do
  if (( i % CIRCLE_NODE_TOTAL == CIRCLE_NODE_INDEX )); then
    MY_TESTS+=("${ALL_TESTS[$i]}")
  fi
done

pytest "${MY_TESTS[@]}"
```

---

## 🎯 Chiến Lược Nâng Cao

### Loại Trừ Test Chậm (Slow Tests)

```bash
# Tách riêng integration tests chậm
circleci tests glob "src/**/*.test.js" | \
grep -v "\.integration\.test\.js" | \   # Bỏ integration tests
circleci tests split --split-by=timings | \
xargs npx jest
```

### Kết Hợp Glob Pattern

```bash
# Tìm nhiều loại test file
circleci tests glob \
  "src/**/*.test.ts" \
  "src/**/*.spec.ts" | \
circleci tests split --split-by=timings
```

### Splitting Với Test Suites Lớn — First-Run Fallback

Lần đầu chạy (chưa có timing data — dữ liệu thời gian), CircleCI tự động fallback về file-based splitting. Sau 2–3 lần chạy, timing sẽ được tích lũy và splitting trở nên chính xác hơn.

---

## 📈 Ước Tính Tiết Kiệm Thời Gian

| Test Suite | Thời Gian Gốc | parallelism: 2 | parallelism: 4 | parallelism: 8 |
| ---------- | ------------- | --------------- | --------------- | --------------- |
| 5 phút | 5 phút | 2.5 phút | 1.5 phút | ~1 phút |
| 20 phút | 20 phút | 10 phút | 5 phút | 2.5–3 phút |
| 60 phút | 60 phút | 30 phút | 15 phút | 7–8 phút |

> **Lưu ý:** Hiệu quả giảm dần vì có overhead khởi động container và phân phối không hoàn toàn đều.

---

## ⚠️ Các Lỗi Thường Gặp

### Lỗi 1: Quên `store_test_results`

```yaml
# ❌ Sai: Timing-based splitting không có data để học
- run:
    command: |
      circleci tests glob "**/*.test.js" | \
      circleci tests split --split-by=timings | \  # Sẽ fallback về file-based
      xargs npx jest
# Không có store_test_results!

# ✅ Đúng: Luôn lưu kết quả
- run: ...  # chạy test
- store_test_results:
    path: test-results/
```

### Lỗi 2: Glob Pattern Không Tìm Thấy File

```bash
# ❌ Quá cụ thể, không có file
circleci tests glob "tests/unit/**/*Test.java"

# ✅ Kiểm tra pattern trước
ls tests/unit/**/*Test.java  # Chạy local để xác nhận
```

### Lỗi 3: Kết Quả Test Trùng Lặp

Khi dùng `circleci tests run` thay vì glob + split, có thể bị trùng nếu cấu hình sai:

```bash
# Luôn verify NODE_INDEX và NODE_TOTAL được set
echo "Container $CIRCLE_NODE_INDEX / $CIRCLE_NODE_TOTAL"
```

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Test splitting là gì và hoạt động ra sao?**

A: Test splitting (phân chia kiểm thử) là kỹ thuật chia bộ test suite thành nhiều phần và chạy song song trên nhiều container. CircleCI dùng `circleci tests split` để phân chia danh sách file test cho mỗi container dựa trên `CIRCLE_NODE_INDEX`. Kết hợp với `store_test_results`, CircleCI học thời gian của từng test để phân chia thông minh hơn (timing-based splitting).

**Q: Khác biệt giữa file-based và timing-based splitting?**

A: File-based chia đều số lượng file — đơn giản nhưng có thể không đều về thời gian nếu một số test chậm hơn nhiều. Timing-based dùng lịch sử thời gian chạy để phân chia sao cho tổng thời gian của mỗi container gần bằng nhau — chính xác hơn nhưng cần vài lần chạy để tích lũy data.

**Q: Khi nào nên dùng test splitting?**

A: Khi test suite chạy > 5 phút thì nên xem xét splitting. Với test suite > 10–15 phút, splitting với parallelism: 4 thường mang lại ROI tốt nhất so với chi phí thêm container.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
