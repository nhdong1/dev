# 📄 Cú Pháp YAML Trong CircleCI Config

> YAML (YAML Ain't Markup Language — YAML Không Phải Ngôn Ngữ Đánh Dấu) là định dạng cấu hình của CircleCI. Hiểu đúng YAML giúp tránh lỗi parse — phân tích cú pháp và viết config rõ ràng, bảo trì được.

---

## 📋 Mục Lục

1. [Cấu Trúc Cơ Bản YAML](#1-cấu-trúc-cơ-bản-yaml)
2. [Các Kiểu Dữ Liệu](#2-các-kiểu-dữ-liệu)
3. [Anchors và Aliases](#3-anchors-và-aliases)
4. [Multi-line Strings](#4-multi-line-strings)
5. [Schema CircleCI config.yml](#5-schema-circleci-configyml)
6. [Lỗi YAML Thường Gặp](#6-lỗi-yaml-thường-gặp)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Cấu Trúc Cơ Bản YAML

### 1.1 Indentation — Thụt Lề

YAML dùng **spaces (dấu cách), không dùng tabs** để thể hiện cấu trúc phân cấp. CircleCI yêu cầu nhất quán — thường dùng 2 spaces.

```yaml
# ✅ Đúng — dùng 2 spaces
jobs:
  my-job:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: echo "Hello"

# ❌ Sai — trộn tabs và spaces
jobs:
	my-job:          # tab ở đây gây lỗi
    docker:
```

### 1.2 Key-Value Pairs — Cặp Khóa-Giá Trị

```yaml
# Cặp khóa-giá trị cơ bản
version: 2.1
name: "My Pipeline"
enabled: true
count: 42

# Giá trị lồng nhau (nested)
job:
  name: build
  docker:
    - image: cimg/node:20.0
```

### 1.3 Lists — Danh Sách

```yaml
# Danh sách dạng block (phổ biến trong CircleCI)
steps:
  - checkout
  - run: npm install
  - run: npm test

# Danh sách dạng flow (inline)
tags: [production, v2, stable]

# Danh sách các objects
docker:
  - image: cimg/node:20.0
    environment:
      NODE_ENV: test
  - image: cimg/postgres:14.0
    environment:
      POSTGRES_DB: testdb
```

### 1.4 Comments — Chú Thích

```yaml
# Đây là comment — dòng chú thích
version: 2.1  # Phiên bản hiện tại khuyến nghị

jobs:
  build:
    # Docker executor với Node.js 20
    docker:
      - image: cimg/node:20.0
```

---

## 2. Các Kiểu Dữ Liệu

### 2.1 Strings — Chuỗi Ký Tự

```yaml
# Unquoted string — chuỗi không dấu ngoặc (phổ biến nhất)
name: my-job

# Double-quoted string — chuỗi ngoặc kép (hỗ trợ escape sequences)
message: "Hello, World!\nNew line"

# Single-quoted string — chuỗi ngoặc đơn (không xử lý escape)
path: 'C:\Users\user\file.txt'

# Chuỗi có ký tự đặc biệt cần dấu ngoặc
command: "echo 'Hello: World'"
filter: "^(main|develop)$"
```

### 2.2 Booleans — Giá Trị Logic

```yaml
# Trong YAML, các giá trị sau đều là boolean
enabled: true       # ✅ Khuyến nghị
disabled: false     # ✅ Khuyến nghị
flag: yes           # Cũng hợp lệ nhưng ít dùng
other: no           # Cũng hợp lệ nhưng ít dùng

# Trong CircleCI
when: on_success    # Không phải boolean, là enum string
no_output_timeout: "10m"  # String, không phải boolean
```

### 2.3 Integers và Floats — Số Nguyên và Số Thực

```yaml
parallelism: 4       # Integer — số nguyên
timeout: 300         # Integer (seconds)
version: 2.1         # Float — số thực (YAML tự nhận diện)
```

### 2.4 Null — Giá Trị Rỗng

```yaml
# Các cách biểu diễn null trong YAML
value: null
value: ~
value:        # Không gán gì cũng là null
```

---

## 3. Anchors và Aliases

**Anchor (`&`)** — Mỏ Neo: đánh dấu một node để tái sử dụng.  
**Alias (`*`)** — Bí Danh: tham chiếu đến anchor đã khai báo.  
**Merge Key (`<<`)** — Khóa Gộp: gộp nội dung của anchor vào object hiện tại.

### 3.1 Anchor Cơ Bản

```yaml
# Khai báo anchor
defaults: &defaults
  docker:
    - image: cimg/node:20.0
  resource_class: medium
  working_directory: ~/app

# Dùng alias để tái sử dụng
jobs:
  build:
    <<: *defaults      # Gộp toàn bộ nội dung defaults vào đây
    steps:
      - checkout
      - run: npm run build

  test:
    <<: *defaults      # Tái sử dụng cùng executor config
    steps:
      - checkout
      - run: npm test
```

### 3.2 Anchor Cho Environment Variables — Biến Môi Trường

```yaml
# Định nghĩa biến môi trường chung
common-env: &common-env
  NODE_ENV: test
  CI: "true"
  LOG_LEVEL: info

jobs:
  unit-test:
    docker:
      - image: cimg/node:20.0
        environment:
          <<: *common-env
          TEST_TYPE: unit

  integration-test:
    docker:
      - image: cimg/node:20.0
        environment:
          <<: *common-env
          TEST_TYPE: integration
          DB_HOST: localhost
```

### 3.3 Anchor Cho Steps Tái Sử Dụng

```yaml
# Anchor cho một nhóm steps cài đặt
install-steps: &install-steps
  - checkout
  - restore_cache:
      keys:
        - node-v1-{{ checksum "package-lock.json" }}
  - run: npm ci
  - save_cache:
      key: node-v1-{{ checksum "package-lock.json" }}
      paths:
        - ~/.npm

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - *install-steps       # Tái sử dụng anchor
      - run: npm run build

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - *install-steps       # Tái sử dụng cùng steps
      - run: npm test
```

> **Lưu ý:** Anchors/Aliases là tính năng YAML thuần, không phải CircleCI-specific. Tuy nhiên, CircleCI khuyến nghị dùng **Reusable Executors** và **Commands** thay thế vì chúng hỗ trợ parameters và được validate đúng hơn.

---

## 4. Multi-line Strings

### 4.1 Block Scalar Literal (`|`) — Giữ Nguyên Xuống Dòng

Ký tự `|` giữ nguyên tất cả newlines — ký tự xuống dòng. Phù hợp cho shell scripts — kịch bản dòng lệnh nhiều dòng.

```yaml
jobs:
  deploy:
    steps:
      - run:
          name: Deploy script
          command: |
            echo "Bắt đầu deploy..."
            npm run build
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              aws s3 sync dist/ s3://my-bucket --delete
              echo "Deploy thành công!"
            else
              echo "Bỏ qua deploy cho nhánh $CIRCLE_BRANCH"
            fi
```

### 4.2 Block Scalar Folded (`>`) — Gộp Dòng Thành Đoạn

Ký tự `>` gộp các dòng liền nhau thành một dòng (thêm space). Xuống dòng trống tạo paragraph mới.

```yaml
jobs:
  notify:
    steps:
      - run:
          name: Thông báo
          command: >
            echo "Pipeline hoàn thành thành công.
            Kết quả đã được lưu vào S3.
            Vui lòng kiểm tra dashboard."
          # Câu trên sẽ thành một dòng dài khi chạy
```

### 4.3 So Sánh `|` và `>`

```yaml
# Với | (literal block) — giữ nguyên xuống dòng
script_literal: |
  line 1
  line 2
  line 3
# Kết quả: "line 1\nline 2\nline 3\n"

# Với > (folded block) — gộp dòng
script_folded: >
  line 1
  line 2
  line 3
# Kết quả: "line 1 line 2 line 3\n"
```

### 4.4 Chomp Indicators — Chỉ Báo Kết Thúc

```yaml
# | hoặc > mặc định giữ 1 newline ở cuối
# |- hoặc >- loại bỏ newline cuối (strip)
# |+ hoặc >+ giữ tất cả newlines cuối (keep)

command: |-
  npm install
  npm test
# Không có newline ở cuối chuỗi
```

---

## 5. Schema CircleCI config.yml

### 5.1 Cấu Trúc Top-level Đầy Đủ

```yaml
version: 2.1                    # BẮT BUỘC

# --- Phần tùy chọn ---

setup: true                     # Bật Dynamic Config — Cấu Hình Động

parameters:                     # Pipeline parameters — Tham Số Pipeline
  deploy-env:
    type: enum
    enum: [staging, production]
    default: staging

orbs:                           # Import orbs — Gói Tích Hợp
  node: circleci/node@5.1.0
  aws-cli: circleci/aws-cli@4.0.0

executors:                      # Môi trường thực thi tái sử dụng
  node-executor:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

commands:                       # Lệnh tái sử dụng
  install-and-cache:
    steps:
      - restore_cache:
          keys:
            - node-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: node-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

jobs:                           # BẮT BUỘC — Định nghĩa công việc
  build:
    executor: node-executor
    steps:
      - checkout
      - install-and-cache
      - run: npm run build

workflows:                      # BẮT BUỘC (trừ Dynamic Config)
  main:
    jobs:
      - build
```

### 5.2 Cấu Trúc Job Chi Tiết

```yaml
jobs:
  job-name:
    # --- Executor (chọn 1 trong 4 loại) ---
    docker:
      - image: <image>
    # hoặc
    machine:
      image: ubuntu-2204:current
    # hoặc
    macos:
      xcode: "15.0.0"
    # hoặc
    executor: <executor-name>    # Tham chiếu executor đã khai báo

    # --- Tùy chọn ---
    resource_class: medium       # CPU/RAM: small, medium, large, xlarge...
    working_directory: ~/app     # Thư mục làm việc mặc định
    parallelism: 4               # Số luồng song song
    shell: /bin/bash             # Shell mặc định
    environment:                 # Biến môi trường ở cấp job
      NODE_ENV: test
    circleci_ip_ranges: false    # Bật IP Ranges — Dải IP cố định
    
    parameters:                  # Job parameters — Tham Số Job
      env:
        type: string
        default: staging

    steps:
      - checkout                 # Built-in step
      - run:
          name: <tên hiển thị>
          command: <shell command>
          when: always           # always | on_success | on_fail
          no_output_timeout: 10m # Timeout nếu không có output
          environment:           # Biến môi trường trong run step này
            MY_VAR: value
      - store_artifacts:
          path: test-results/
          destination: test-results
      - store_test_results:
          path: test-results/
```

### 5.3 Cấu Trúc Workflow Chi Tiết

```yaml
workflows:
  workflow-name:
    # Trigger — Kích Hoạt theo lịch (tùy chọn)
    triggers:
      - schedule:
          cron: "0 8 * * 1-5"   # 8 giờ sáng, thứ 2 đến thứ 6
          filters:
            branches:
              only:
                - main

    jobs:
      - job-name-1
      - job-name-2:
          requires:
            - job-name-1         # Chạy sau job-name-1
          filters:
            branches:
              only: main         # Chỉ chạy trên nhánh main
            tags:
              only: /^v.*/       # Chỉ chạy khi có tag v*
          context:
            - my-context         # Dùng Context — Ngữ Cảnh
          matrix:                # Ma trận biến thể
            parameters:
              node-version: ["18", "20", "22"]
      - approval-job:
          type: approval         # Manual gate — Cổng Duyệt Thủ Công
          requires:
            - job-name-2
```

---

## 6. Lỗi YAML Thường Gặp

### 6.1 Tab Thay Vì Spaces

```yaml
# ❌ Lỗi: sử dụng tab
jobs:
	build:          # Tab gây lỗi "found character that cannot start any token"
    steps:
      - checkout

# ✅ Đúng: dùng 2 spaces
jobs:
  build:
    steps:
      - checkout
```

### 6.2 Colon Trong String Không Có Dấu Ngoặc

```yaml
# ❌ Lỗi: colon trong value
command: echo Hello: World    # YAML hiểu "World" là key mới

# ✅ Đúng: đặt trong dấu ngoặc
command: "echo Hello: World"
command: 'echo Hello: World'
```

### 6.3 Giá Trị Boolean Không Mong Muốn

```yaml
# ❌ Nguy hiểm: YAML 1.1 hiểu "yes/no/on/off" là boolean
environment:
  DEBUG: yes      # Thành true, không phải string "yes"
  FEATURE: off    # Thành false, không phải string "off"

# ✅ Đúng: đặt trong dấu ngoặc kép
environment:
  DEBUG: "yes"
  FEATURE: "off"
```

### 6.4 Indentation Không Nhất Quán

```yaml
# ❌ Lỗi: indentation không đồng đều
steps:
  - checkout
    - run: npm install    # Thụt lề sai

# ✅ Đúng
steps:
  - checkout
  - run: npm install
```

### 6.5 Thiếu Dấu `-` Trong List

```yaml
# ❌ Lỗi: docker images phải là list
docker:
  image: cimg/node:20.0    # Thiếu dấu -

# ✅ Đúng
docker:
  - image: cimg/node:20.0
```

### 6.6 String Có Ký Tự Đặc Biệt

```yaml
# ❌ Lỗi: các ký tự {, }, [, ], ,, #, |, >, ! cần được quote
name: {build-job}
filter: [main, develop]

# ✅ Đúng
name: "{build-job}"
filter: "[main, develop]"

# Hoặc dùng block syntax
filter: |
  [main, develop]
```

---

## 7. Câu Hỏi Phỏng Vấn

### Câu hỏi 1: YAML anchors có tác dụng gì trong CircleCI config?

**Trả lời:**
YAML anchors (`&`) và aliases (`*`) cho phép tái sử dụng một đoạn cấu hình ở nhiều nơi, giảm duplication — trùng lặp. Ví dụ, khai báo executor config một lần rồi dùng `<<: *anchor-name` để gộp vào nhiều jobs.

Tuy nhiên, trong CircleCI version 2.1, cách được khuyến nghị hơn là dùng **Reusable Executors** và **Commands** vì chúng được validate chặt chẽ, hỗ trợ parameters và rõ ràng hơn khi đọc.

---

### Câu hỏi 2: Sự khác biệt giữa `|` và `>` trong YAML?

**Trả lời:**
- `|` (literal block scalar — block chữ nguyên văn): giữ nguyên tất cả newlines. Dùng cho shell scripts nhiều dòng.
- `>` (folded block scalar — block chữ gấp): gộp các dòng liền nhau thành một dòng, chỉ giữ newline khi có dòng trống. Dùng cho chuỗi văn bản dài.

Trong CircleCI, hầu như luôn dùng `|` cho `command:` vì shell scripts cần giữ nguyên xuống dòng.

---

### Câu hỏi 3: Làm sao validate file config.yml trước khi push?

**Trả lời:**
```bash
# Dùng CircleCI CLI
circleci config validate

# Xem config sau khi orbs được expand
circleci config process .circleci/config.yml

# Chạy thử job cục bộ
circleci local execute --job <job-name>
```

Ngoài ra, có thể dùng online validator tại `https://circleci.com/developer/` hoặc cài VS Code extension hỗ trợ YAML schema của CircleCI.

---

**Tiếp theo:** [2-jobs-and-steps.md](2-jobs-and-steps.md) — Jobs và Steps
