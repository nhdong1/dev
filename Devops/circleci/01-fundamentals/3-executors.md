# 3 — Executors — Môi Trường Thực Thi

> Executor — Môi Trường Thực Thi xác định nơi mà Job của bạn chạy. CircleCI hỗ trợ bốn loại: Docker, Machine (VM — Virtual Machine), macOS và Windows. Chọn đúng Executor ảnh hưởng trực tiếp đến tốc độ build, chi phí và khả năng của pipeline.

---

## 📚 Mục Lục

1. [Tổng Quan Bốn Loại Executor](#tổng-quan-bốn-loại-executor)
2. [Docker Executor](#docker-executor)
3. [Machine Executor](#machine-executor)
4. [macOS Executor](#macos-executor)
5. [Windows Executor](#windows-executor)
6. [So Sánh và Lựa Chọn](#so-sánh-và-lựa-chọn)
7. [Reusable Executors — Executor Tái Sử Dụng](#reusable-executors--executor-tái-sử-dụng)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Bốn Loại Executor

```
┌──────────────────────────────────────────────────────────────────┐
│                     CircleCI Executors                           │
├─────────────┬──────────────┬────────────┬────────────────────────┤
│   Docker    │   Machine    │   macOS    │       Windows          │
│  (Container)│  (Linux VM)  │  (Mac VM)  │   (Windows VM)         │
├─────────────┼──────────────┼────────────┼────────────────────────┤
│ Khởi động   │              │            │                        │
│  nhanh nhất │  ~30-60s     │  ~1-2 phút │  ~1-2 phút             │
│  (~10-20s)  │              │            │                        │
├─────────────┼──────────────┼────────────┼────────────────────────┤
│ Web, API,   │ Docker-in-   │ iOS, macOS │ .NET, WPF,             │
│ Build apps  │ Docker, kVM  │ native app │ Windows-only tools     │
└─────────────┴──────────────┴────────────┴────────────────────────┘
```

---

## Docker Executor

### Cách Hoạt Động

Docker Executor chạy Job trong một Docker container — bộ chứa. CircleCI tạo container từ image bạn chỉ định, chạy các Step bên trong, rồi xóa container sau khi xong.

```
CircleCI Host
┌─────────────────────────────────────────┐
│  Docker Daemon                          │
│  ┌──────────────────────────────────┐   │
│  │  Primary Container               │   │
│  │  (image: cimg/node:20.0)         │   │
│  │  - Chạy tất cả Step của Job      │   │
│  │  - Mount: checkout code          │   │
│  ├──────────────────────────────────┤   │
│  │  Service Container (tùy chọn)    │   │
│  │  (image: postgres:15)            │   │
│  │  - Database cho integration test │   │
│  │  - Accessible tại localhost:5432 │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Cú Pháp Cơ Bản

```yaml
jobs:
  test:
    docker:
      - image: cimg/node:20.0           # Primary image — ảnh chính
    steps:
      - checkout
      - run: npm test
```

### Multi-Container Setup — Nhiều Container

```yaml
jobs:
  integration-test:
    docker:
      # Primary container — nơi Steps chạy
      - image: cimg/python:3.11
        environment:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      # Service containers — chạy song song với primary
      - image: postgres:15
        environment:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: testdb

      - image: redis:7-alpine

    steps:
      - checkout
      - run: pip install -r requirements.txt
      - run:
          name: Chờ PostgreSQL sẵn sàng
          command: |
            dockerize -wait tcp://localhost:5432 -timeout 1m
      - run: pytest tests/integration/
```

> **dockerize** là tool giúp chờ service khác sẵn sàng trước khi chạy test — tránh race condition — điều kiện tranh đua.

### CircleCI Convenience Images — Ảnh Tiện Ích

CircleCI cung cấp các `cimg/*` images được tối ưu cho CI:

```yaml
# Các cimg/* images phổ biến
cimg/node:20.0          # Node.js
cimg/python:3.11        # Python
cimg/ruby:3.2           # Ruby
cimg/go:1.21            # Go
cimg/openjdk:21.0       # Java
cimg/rust:1.75          # Rust
cimg/base:stable        # Base image, không có runtime cụ thể
cimg/aws:2023.09        # AWS CLI
cimg/postgres:15.0      # PostgreSQL (dùng làm service)
```

**Lợi thế của `cimg/*` so với official images:**
- Pre-installed CI tools: git, ssh, dockerize, jq
- User có quyền sudo mà không cần password
- Tương thích với CircleCI features như SSH debugging
- Được cập nhật thường xuyên và test kỹ

### Hạn Chế Docker Executor

```
❌ Không thể chạy Docker commands bên trong (Docker-in-Docker cần setup phức tạp)
❌ Không có systemd — trình quản lý dịch vụ
❌ Kernel bị chia sẻ với host → không test kernel features
❌ Không thể mount /dev hoặc /proc tùy ý
```

---

## Machine Executor

### Cách Hoạt Động

Machine Executor cung cấp một Linux VM — Virtual Machine (Máy Ảo) đầy đủ. Bạn có toàn quyền kiểm soát: cài bất kỳ package nào, chạy Docker daemon, mount bất kỳ thứ gì.

```
CircleCI Infrastructure
┌─────────────────────────────────────┐
│  Dedicated Linux VM (Ubuntu 20.04)  │
│  ┌─────────────────────────────┐    │
│  │  Full OS Environment        │    │
│  │  - Docker daemon chạy sẵn   │    │
│  │  - systemd available        │    │
│  │  - /dev access              │    │
│  │  - sudo không cần password  │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

### Cú Pháp

```yaml
jobs:
  docker-build:
    machine:
      image: ubuntu-2204:current    # Ubuntu 22.04 LTS, luôn cập nhật
      docker_layer_caching: true    # DLC — Docker Layer Caching: cache Docker layers
    resource_class: medium          # Tương tự Docker executor
    steps:
      - checkout
      - run:
          name: Build Docker image
          command: docker build -t myapp:$CIRCLE_SHA1 .
      - run:
          name: Chạy tests trong container
          command: docker run --rm myapp:$CIRCLE_SHA1 npm test
      - run:
          name: Push lên ECR
          command: |
            aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
            docker push $ECR_REGISTRY/myapp:$CIRCLE_SHA1
```

### Docker Layer Caching — DLC — Bộ Đệm Layer Docker

```yaml
machine:
  image: ubuntu-2204:current
  docker_layer_caching: true    # Cache layers Docker → build nhanh hơn
```

DLC lưu các Docker layer từ build trước. Khi Dockerfile ít thay đổi, các layer unchanged được reuse — tái sử dụng, giảm đáng kể thời gian build.

```
Build 1 (không có cache):
  Step 1: FROM node:20      → 200MB download (60s)
  Step 2: COPY package.json → 0.1s
  Step 3: RUN npm install   → 120s
  Step 4: COPY src/         → 1s
  Tổng: ~3 phút

Build 2 (có DLC, chỉ src/ thay đổi):
  Step 1: FROM node:20      → CACHED (0s)
  Step 2: COPY package.json → CACHED (0s)
  Step 3: RUN npm install   → CACHED (0s)
  Step 4: COPY src/         → 1s
  Tổng: ~10s
```

### Các Image Machine Phổ Biến

```yaml
# Ubuntu
ubuntu-2204:current   # Ubuntu 22.04 LTS (khuyến nghị)
ubuntu-2004:current   # Ubuntu 20.04 LTS

# Android (có Android SDK sẵn)
android:2023.11.1

# GPU (cho ML workloads — tải công việc Machine Learning)
gpu.nvidia.small      # NVIDIA T4 GPU
```

---

## macOS Executor

### Cách Hoạt Động

macOS Executor chạy Job trên máy ảo macOS thực sự (không phải container). Được sử dụng chủ yếu để build và test ứng dụng iOS, macOS, và các ứng dụng cross-platform cần môi trường macOS.

```yaml
jobs:
  ios-build:
    macos:
      xcode: "15.2.0"    # Phiên bản Xcode cụ thể
    resource_class: macos.m1.medium.gen1    # Apple Silicon M1
    steps:
      - checkout
      - run:
          name: Cài CocoaPods dependencies
          command: pod install
      - run:
          name: Build và test iOS app
          command: |
            xcodebuild \
              -workspace MyApp.xcworkspace \
              -scheme MyApp \
              -sdk iphonesimulator \
              -destination 'platform=iOS Simulator,name=iPhone 15' \
              clean build test
      - store_test_results:
          path: test-results/
```

### Resource Classes macOS

```
macos.m1.medium.gen1    → Apple M1, 4 vCPU, 8 GB RAM (phổ biến nhất)
macos.m1.large.gen1     → Apple M1, 8 vCPU, 12 GB RAM
macos.m2.medium.gen2    → Apple M2, 4 vCPU, 8 GB RAM
```

### Khi Nào Dùng macOS Executor

```
✅ Build iOS app (Xcode, Swift, Objective-C)
✅ Build macOS native app
✅ Test app trên iOS Simulator — Trình Giả Lập iOS
✅ Ứng dụng React Native / Flutter cần build iOS
✅ Tools chỉ có trên macOS (codesign, notarytool)

❌ Không dùng cho: web app, backend service, Python/Node CLI tools
   (dùng Docker executor sẽ nhanh và rẻ hơn nhiều)
```

---

## Windows Executor

### Cú Pháp

Windows Executor cần dùng thông qua Orb — Gói Tích Hợp `circleci/windows`:

```yaml
version: 2.1

orbs:
  win: circleci/windows@5.0.0    # Import Windows orb

jobs:
  dotnet-build:
    executor:
      name: win/default            # Windows Server 2022
      size: medium                 # 4 vCPU, 15 GB RAM
    steps:
      - checkout
      - run:
          name: Build .NET app
          shell: powershell.exe    # Dùng PowerShell
          command: |
            dotnet restore
            dotnet build --configuration Release
            dotnet test --logger "trx"
      - store_test_results:
          path: TestResults/
```

### Shells Hỗ Trợ Trên Windows

```yaml
# PowerShell (mặc định — default)
shell: powershell.exe

# CMD — Command Prompt
shell: cmd.exe

# Bash (Git Bash)
shell: bash.exe
```

### Khi Nào Dùng Windows Executor

```
✅ Build .NET / C# applications
✅ Build WPF — Windows Presentation Foundation apps
✅ Test trên Windows-specific environment
✅ Build game với Unity (Windows build target)
✅ Test PowerShell scripts

❌ Không dùng cho: Linux-based apps, containerized workloads
```

---

## So Sánh và Lựa Chọn

### Bảng So Sánh Đầy Đủ

| Tiêu Chí | Docker | Machine | macOS | Windows |
| --------- | ------ | ------- | ----- | ------- |
| **Khởi động** | ~10-20s | ~30-60s | ~60-120s | ~60-120s |
| **Môi trường** | Container | Linux VM | macOS VM | Windows VM |
| **Docker trong Docker** | Khó (setup phức tạp) | Sẵn sàng | Không phổ biến | Hỗ trợ |
| **systemd** | Không | Có | Không | Không |
| **Chi phí credits** | Thấp nhất | Trung bình | Cao nhất | Cao |
| **Cô lập** | Process | VM đầy đủ | VM đầy đủ | VM đầy đủ |
| **Phù hợp nhất** | Web/API/CLI | Docker build | iOS/macOS | .NET/Windows |

### Decision Tree — Cây Quyết Định

```
Bạn cần build gì?
    │
    ├─► iOS / macOS app? ─────────────────────────► macOS Executor
    │
    ├─► .NET / WPF / Windows app? ───────────────► Windows Executor
    │
    ├─► Cần build Docker image?
    │       │
    │       ├─► Có, và cần push lên registry? ───► Machine Executor (với DLC)
    │       │
    │       └─► Không phức tạp, chỉ dùng orb? ──► Docker Executor (circleci/docker orb)
    │
    └─► Web app, API, CLI, backend? ─────────────► Docker Executor (nhanh + rẻ nhất)
```

### Ví Dụ Thực Tế Chọn Executor

```
Dự án: E-commerce backend (Node.js)
→ Docker Executor (cimg/node:20.0)
→ Service container PostgreSQL cho integration tests
→ Lý do: Nhanh, rẻ, đủ cho mọi nhu cầu

Dự án: Mobile app (React Native)
→ iOS build: macOS Executor (Xcode 15.2)
→ Android build: Machine Executor (android:2023.11.1)
→ Backend API test: Docker Executor
→ Lý do: Mỗi target platform cần executor riêng

Dự án: Internal DevOps tool (cần build Docker image)
→ Machine Executor với docker_layer_caching: true
→ Lý do: Cần Docker daemon đầy đủ để build/push image
```

---

## Reusable Executors — Executor Tái Sử Dụng

Khi nhiều Job dùng cùng executor, định nghĩa một lần và tái sử dụng:

```yaml
version: 2.1

# Định nghĩa executor ở top level
executors:
  node-20:
    docker:
      - image: cimg/node:20.0
    resource_class: medium
    working_directory: ~/project
    environment:
      NODE_ENV: test

  node-20-large:
    docker:
      - image: cimg/node:20.0
    resource_class: large     # Cho jobs nặng hơn
    working_directory: ~/project

jobs:
  test:
    executor: node-20         # Tái sử dụng executor
    steps:
      - checkout
      - run: npm test

  build:
    executor: node-20         # Cùng executor
    steps:
      - checkout
      - run: npm run build

  performance-test:
    executor: node-20-large   # Executor lớn hơn cho performance test
    steps:
      - checkout
      - run: npm run perf-test
```

**Lợi ích:**
- DRY — Don't Repeat Yourself — Không Lặp Lại: thay đổi một chỗ, áp dụng toàn bộ
- Dễ update phiên bản Node.js/Python/... cho tất cả jobs
- Rõ ràng, dễ đọc config

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào dùng Machine executor thay vì Docker executor?

**Trả lời mẫu:**

> Machine executor phù hợp khi Job cần Docker daemon đầy đủ để build và push Docker images — đây là use case phổ biến nhất. Cũng dùng Machine khi cần systemd để khởi chạy services, khi test cần quyền truy cập kernel-level, hoặc khi có yêu cầu về cô lập bảo mật cao hơn giữa các Job.
>
> Docker executor phù hợp hơn cho hầu hết trường hợp còn lại: khởi động nhanh hơn (~10s vs ~30-60s), chi phí thấp hơn, và tích hợp tốt với CircleCI convenience images.

### Câu 2: Docker Executor có thể chạy Docker commands không?

**Trả lời mẫu:**

> Mặc định thì khó vì Docker Executor không có Docker daemon riêng. Có ba cách giải quyết: (1) dùng Machine Executor thay thế — đơn giản và đáng tin cậy nhất; (2) dùng Remote Docker — một Docker daemon riêng được CircleCI cấp phát, kích hoạt bằng `setup_remote_docker`, nhưng không chia sẻ network với primary container; (3) dùng orb `circleci/docker` để đơn giản hóa. Với production pipelines, tôi thường chọn Machine Executor với `docker_layer_caching: true` để tận dụng layer cache.

### Câu 3: DLC — Docker Layer Caching là gì và khi nào nên dùng?

**Trả lời mẫu:**

> DLC — Docker Layer Caching lưu lại các Docker layers từ build trước. Khi build lại, nếu layer nào chưa thay đổi (ví dụ `FROM` và `RUN npm install`), CircleCI reuse layer cũ thay vì rebuild. Kết quả là build nhanh hơn đáng kể khi chỉ có source code thay đổi nhưng dependencies không đổi.
>
> Nên dùng DLC khi: build Docker image thường xuyên, Dockerfile có nhiều heavy layers (npm install, apt-get), và chi phí DLC (credits) thấp hơn thời gian tiết kiệm được.

---

## 🔗 Đọc Tiếp

- [4-resource-classes.md](4-resource-classes.md) — CPU/RAM và chiến lược tối ưu chi phí
- [../02-configuration/README.md](../02-configuration/README.md) — Cấu hình YAML đầy đủ
- [../07-integration/1-docker-build-push.md](../07-integration/1-docker-build-push.md) — Build và push Docker image

---

**Thời Gian Đọc:** 45–60 phút  
**Cập Nhật:** 2026-05-18
