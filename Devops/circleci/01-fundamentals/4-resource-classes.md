# 4 — Resource Classes — Lớp Tài Nguyên

> Resource Class — Lớp Tài Nguyên xác định lượng CPU và RAM được cấp cho mỗi Job. Chọn đúng resource class giúp cân bằng giữa tốc độ build và chi phí, tránh OOM (Out Of Memory — Hết Bộ Nhớ) và tối ưu credit CircleCI.

---

## 📚 Mục Lục

1. [Resource Class là gì?](#resource-class-là-gì)
2. [Docker Resource Classes](#docker-resource-classes)
3. [Machine Resource Classes](#machine-resource-classes)
4. [macOS Resource Classes](#macos-resource-classes)
5. [Windows Resource Classes](#windows-resource-classes)
6. [Chiến Lược Tối Ưu Chi Phí](#chiến-lược-tối-ưu-chi-phí)
7. [Cấu Hình Trong CircleCI](#cấu-hình-trong-circleci)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Resource Class là gì?

Mỗi Job trong CircleCI cần tài nguyên tính toán để chạy. Resource Class là preset — cấu hình sẵn các mức CPU và RAM khác nhau:

```
┌────────────────────────────────────────────────────────┐
│              Resource Class Spectrum                   │
│                                                        │
│  small   medium   medium+  large   xlarge  2xlarge     │
│  │        │        │        │        │        │        │
│  1 vCPU  2 vCPU   4 vCPU   4 vCPU  8 vCPU  16 vCPU   │
│  2 GB    4 GB     8 GB     8 GB    16 GB    32 GB      │
│                                                        │
│  Rẻ ◄────────────────────────────────────► Đắt       │
│  Chậm ◄──────────────────────────────────► Nhanh     │
└────────────────────────────────────────────────────────┘
```

**Chi phí đo bằng Credits — Tín Dụng:**
- CircleCI tính phí theo số credits tiêu thụ
- Mỗi resource class tiêu thụ số credits/phút khác nhau
- Free tier có giới hạn credits/tuần

---

## Docker Resource Classes

### Bảng Resource Class Docker / Linux

| Resource Class | vCPU | RAM | Credits/phút | Dùng Cho |
| -------------- | ---- | --- | ------------ | -------- |
| `small` | 1 | 2 GB | 5 | Script đơn giản, lint |
| `medium` | 2 | 4 GB | 10 | **Mặc định, phổ biến nhất** |
| `medium+` | 3 | 6 GB | 15 | Test suite vừa |
| `large` | 4 | 8 GB | 20 | Build nặng, test song song |
| `xlarge` | 8 | 16 GB | 40 | Compilation lớn, nhiều parallelism |
| `2xlarge` | 16 | 32 GB | 80 | Monorepo build, ML training |
| `2xlarge+` | 20 | 40 GB | 100 | Workload cực nặng |

### Ví Dụ Thực Tế

```yaml
jobs:
  lint:
    docker:
      - image: cimg/node:20.0
    resource_class: small     # 1 vCPU, 2 GB — đủ để chạy ESLint
    steps:
      - checkout
      - run: npm run lint

  unit-test:
    docker:
      - image: cimg/node:20.0
    resource_class: medium    # 2 vCPU, 4 GB — Jest unit tests
    parallelism: 4
    steps:
      - checkout
      - run: npm test

  integration-test:
    docker:
      - image: cimg/node:20.0
        auth:
          username: $DOCKERHUB_USER
          password: $DOCKERHUB_TOKEN
      - image: postgres:15
      - image: redis:7
    resource_class: large     # 4 vCPU, 8 GB — cần RAM cho 3 containers
    steps:
      - checkout
      - run: npm run test:integration

  webpack-build:
    docker:
      - image: cimg/node:20.0
    resource_class: xlarge    # 8 vCPU, 16 GB — Webpack build song song
    steps:
      - checkout
      - run: npm run build:production
```

---

## Machine Resource Classes

### Bảng Resource Class Machine (Linux VM)

| Resource Class | vCPU | RAM | Credits/phút | Dùng Cho |
| -------------- | ---- | --- | ------------ | -------- |
| `medium` | 2 | 7.5 GB | 10 | Docker build nhỏ |
| `large` | 4 | 15 GB | 20 | Docker build trung bình |
| `xlarge` | 8 | 32 GB | 40 | Docker build lớn |
| `2xlarge` | 16 | 64 GB | 80 | Monorepo Docker build |
| `gpu.nvidia.small` | 4 vCPU + 1 NVIDIA T4 | 15 GB | 80 | ML training |
| `gpu.nvidia.medium` | 8 vCPU + 1 NVIDIA T4 | 30 GB | 160 | ML training nặng |
| `arm.medium` | 2 ARM vCPU | 8 GB | 10 | ARM builds |
| `arm.large` | 4 ARM vCPU | 16 GB | 20 | ARM builds lớn |

### ARM Resource Classes

Kể từ 2023, CircleCI hỗ trợ ARM architecture — kiến trúc ARM:

```yaml
jobs:
  build-arm:
    machine:
      image: ubuntu-2204:current
    resource_class: arm.medium    # ARM processor — bộ vi xử lý ARM
    steps:
      - checkout
      - run:
          name: Build Docker image cho ARM
          command: docker buildx build --platform linux/arm64 -t myapp:arm64 .
```

---

## macOS Resource Classes

### Bảng Resource Class macOS

| Resource Class | CPU | RAM | Credits/phút | Chip |
| -------------- | --- | --- | ------------ | ---- |
| `macos.m1.medium.gen1` | 4 vCPU | 8 GB | 100 | Apple M1 |
| `macos.m1.large.gen1` | 8 vCPU | 12 GB | 200 | Apple M1 |
| `macos.m2.medium.gen2` | 4 vCPU | 8 GB | 150 | Apple M2 |
| `macos.m2.large.gen2` | 8 vCPU | 12 GB | 250 | Apple M2 |

> macOS resource classes đắt hơn đáng kể so với Docker/Machine. Chỉ dùng cho iOS/macOS builds.

```yaml
jobs:
  ios-test:
    macos:
      xcode: "15.2.0"
    resource_class: macos.m1.medium.gen1    # Apple M1, phù hợp hầu hết iOS projects
    steps:
      - checkout
      - run: pod install
      - run: xcodebuild test -scheme MyApp
```

---

## Windows Resource Classes

### Bảng Resource Class Windows

| Resource Class | vCPU | RAM | Credits/phút |
| -------------- | ---- | --- | ------------ |
| `windows.medium` (default) | 4 | 15 GB | 40 |
| `windows.large` | 8 | 30 GB | 80 |
| `windows.xlarge` | 16 | 60 GB | 160 |
| `windows.2xlarge` | 32 | 128 GB | 320 |

```yaml
version: 2.1

orbs:
  win: circleci/windows@5.0.0

jobs:
  build:
    executor:
      name: win/default
      size: medium          # windows.medium: 4 vCPU, 15 GB
    steps:
      - checkout
      - run:
          shell: powershell.exe
          command: dotnet build
```

---

## Chiến Lược Tối Ưu Chi Phí

### Nguyên Tắc Cơ Bản

```
Thời gian Job × Credits/phút = Tổng Credits Tiêu Thụ

Ví dụ:
  medium (10 credits/phút) × 10 phút  = 100 credits
  large  (20 credits/phút) × 5 phút   = 100 credits (xây dựng nhanh hơn 2x)
  xlarge (40 credits/phút) × 3 phút   = 120 credits (xây dựng nhanh hơn 3x, đắt hơn 20%)
```

**Quy tắc:** Tăng resource class **không nhất thiết** tăng chi phí nếu nó giảm thời gian tương ứng.

### Quy Trình Tối Ưu

#### Bước 1: Đo Thời Gian Build Hiện Tại

Xem CircleCI Insights — Thống Kê để biết từng job mất bao lâu:

```
Test job: 15 phút — medium (10 cred/min) → 150 credits
Build job: 8 phút — medium (10 cred/min) → 80 credits
```

#### Bước 2: Xác Định Bottleneck — Điểm Nghẽn

```
Câu hỏi:
✓ Job nào chạy lâu nhất? → Đó là ứng viên để optimize
✓ Job đó bị giới hạn bởi CPU hay RAM?
✓ CPU usage thường xuyên 100%? → Cần nhiều CPU hơn
✓ OOM errors? → Cần nhiều RAM hơn
✓ Job chạy song song được không? → Tăng parallelism thay vì resource class
```

#### Bước 3: Chiến Lược Tiết Kiệm

```yaml
# Chiến lược 1: Small resource cho simple jobs
lint:
  resource_class: small   # Lint không cần nhiều tài nguyên

# Chiến lược 2: Parallelism thay vì resource class lớn hơn
test:
  resource_class: medium
  parallelism: 4          # 4 × medium rẻ hơn 1 × 2xlarge
                          # nhưng test kết thúc nhanh hơn

# Chiến lược 3: Chỉ dùng large cho jobs thực sự cần
webpack-build:
  resource_class: large   # Webpack hưởng lợi từ nhiều CPU
  # Đo lường: medium = 20 phút, large = 10 phút
  # medium: 200 credits, large: 200 credits → Hòa vốn, chọn large (nhanh hơn)
```

#### Bước 4: So Sánh Thực Tế

```
Pipeline trước khi optimize:
  lint:             small  ×  3 min = 15 credits
  test (no split):  medium × 20 min = 200 credits
  build:            medium × 10 min = 100 credits
  Tổng: 315 credits, tổng thời gian: 33 phút

Pipeline sau khi optimize:
  lint:             small  ×  3 min = 15 credits
  test (split 4×):  medium ×  5 min × 4 = 200 credits (4 containers song song)
  build:            large  ×  5 min × 20 = 100 credits (nhanh gấp đôi)
  Tổng: 315 credits, tổng thời gian: 13 phút (giảm 60%)
```

### OOM — Out Of Memory — Hết Bộ Nhớ: Dấu Hiệu Cần Tăng RAM

```bash
# Dấu hiệu OOM trong logs
Killed                   # Process bị kill do hết RAM
exit code 137            # OOM killer — trình dọn dẹp OOM của Linux
ENOMEM                   # Error: not enough memory
Cannot allocate memory   # Lỗi rõ ràng
```

Khi thấy dấu hiệu OOM:
```yaml
# Tăng resource class (thêm RAM)
resource_class: large   # medium → large: 4 GB → 8 GB
```

### Tối Ưu Chi Phí macOS (Executor Đắt Nhất)

```yaml
# Chỉ chạy macOS build khi thực sự cần
ios-build:
  macos:
    xcode: "15.2.0"
  resource_class: macos.m1.medium.gen1
  steps:
    - checkout
    - restore_cache:
        keys:
          - pods-v1-{{ checksum "Podfile.lock" }}
    - run: pod install
    - save_cache:
        key: pods-v1-{{ checksum "Podfile.lock" }}
        paths:
          - Pods/
    # Cache CocoaPods để tránh tải lại mỗi build
    # → Tiết kiệm 2-5 phút × 100-150 credits/phút
```

---

## Cấu Hình Trong CircleCI

### Xem Resource Class Đang Dùng

CircleCI UI — Giao Diện Người Dùng hiển thị resource class trong mỗi job run:

```
Job: test
  Resource Class: medium
  Duration: 8m 23s
  Credits: 84
```

### Ví Dụ Config Đa Dạng Resource Class

```yaml
version: 2.1

executors:
  node-small:
    docker:
      - image: cimg/node:20.0
    resource_class: small

  node-medium:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

  node-large:
    docker:
      - image: cimg/node:20.0
    resource_class: large

jobs:
  # Lint và format check — nhẹ, dùng small
  lint:
    executor: node-small
    steps:
      - checkout
      - run: npm run lint
      - run: npm run format:check

  # Unit tests — dùng medium với parallelism
  unit-tests:
    executor: node-medium
    parallelism: 4
    steps:
      - checkout
      - run:
          command: |
            circleci tests glob "**/*.test.js" | \
            circleci tests split --split-by=timings | \
            xargs npx jest

  # Build production — Webpack cần nhiều CPU
  build:
    executor: node-large
    steps:
      - checkout
      - run: npm run build:prod
      - persist_to_workspace:
          root: .
          paths: [dist/]

workflows:
  ci:
    jobs:
      - lint
      - unit-tests
      - build:
          requires: [unit-tests]
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Làm thế nào để quyết định chọn resource class nào?

**Trả lời mẫu:**

> Tôi bắt đầu với `medium` làm baseline cho hầu hết jobs. Sau đó dùng CircleCI Insights để xem thời gian build và tìm bottleneck. Nếu test suite chạy 20 phút, tôi sẽ thử song song hóa với `parallelism: 4` trên `medium` thay vì tăng lên `xlarge` — vì song song hóa thường hiệu quả và rẻ hơn.
>
> Tăng resource class khi: thấy OOM (exit code 137), CPU liên tục 100% và không thể song song hóa, hoặc khi build là compilation nặng (TypeScript, Rust, C++) hưởng lợi từ nhiều CPU.

### Câu 2: Bạn đã từng tối ưu chi phí CI/CD như thế nào?

**Trả lời mẫu:**

> Tại dự án X, pipeline CI mất 25 phút và tiêu thụ ~600 credits mỗi run. Tôi phân tích qua Insights và phát hiện: job test mất 18 phút trên `medium`, job build mất 7 phút. Giải pháp: (1) Thêm `parallelism: 4` cho test job → giảm từ 18 xuống 5 phút, cùng tổng credits nhưng nhanh gấp 3x; (2) Tăng build từ `medium` lên `large` → từ 7 xuống 3 phút, credits tương đương. Kết quả: pipeline từ 25 phút → 8 phút, credits giảm 15%, developer productivity tăng rõ rệt.

### Câu 3: Khi nào nên tăng resource class, khi nào nên tăng parallelism?

**Trả lời mẫu:**

> **Tăng resource class** khi công việc không thể song song hóa: compilation, webpack bundling, single-threaded operations. Cũng tăng khi bị OOM.
>
> **Tăng parallelism** khi có nhiều tests độc lập có thể chạy song song. CircleCI `circleci tests split` phân phối tests đều giữa các container, giảm wall-clock time — thời gian thực trong khi tổng credits giữ nguyên (hoặc tăng nhẹ do overhead).
>
> Trong thực tế, test splitting + parallelism thường ROI — Return on Investment — tỷ suất hoàn vốn tốt hơn việc tăng resource class cho test jobs.

---

## 🔗 Đọc Tiếp

- [5-circleci-vs-alternatives.md](5-circleci-vs-alternatives.md) — So sánh công cụ CI/CD
- [../05-optimization/4-parallelism.md](../05-optimization/4-parallelism.md) — Parallelism và test splitting chi tiết
- [../05-optimization/5-pipeline-insights.md](../05-optimization/5-pipeline-insights.md) — Phân tích và tối ưu pipeline

---

**Thời Gian Đọc:** 30 phút  
**Cập Nhật:** 2026-05-18
