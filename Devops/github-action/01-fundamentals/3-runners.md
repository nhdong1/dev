# Runners — Máy Chạy Workflow

> Runner (Máy Chạy) là máy chủ thực thi các jobs trong workflow. Hiểu rõ runners giúp bạn chọn đúng môi trường, tối ưu chi phí, và thiết kế pipeline phù hợp với yêu cầu bảo mật.

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [GitHub-hosted Runners](#github-hosted-runners)
3. [Self-hosted Runners](#self-hosted-runners)
4. [So Sánh Chi Tiết](#so-sánh-chi-tiết)
5. [Larger Runners](#larger-runners)
6. [Container Jobs](#container-jobs)
7. [Runner Labels](#runner-labels)
8. [Best Practices](#best-practices)

---

## Tổng Quan

```
GitHub Actions Jobs
         │
         ├── GitHub-hosted Runners
         │   ├── ubuntu-latest (Ubuntu 22.04/24.04)
         │   ├── windows-latest (Windows Server 2022)
         │   └── macos-latest (macOS 14 Sonoma)
         │
         └── Self-hosted Runners
             ├── Máy vật lý on-premise
             ├── VM trong cloud (EC2, GCE, Azure VM...)
             └── Pod trong Kubernetes (ARC — Actions Runner Controller)
```

---

## GitHub-hosted Runners

### Các Loại Runner Có Sẵn

| Label | OS | CPU | RAM | Storage | Giá (public repos) |
|---|---|---|---|---|---|
| `ubuntu-latest` | Ubuntu 22.04/24.04 | 4 core | 16 GB | 14 GB SSD | Miễn phí |
| `ubuntu-22.04` | Ubuntu 22.04 | 4 core | 16 GB | 14 GB SSD | Miễn phí |
| `ubuntu-24.04` | Ubuntu 24.04 | 4 core | 16 GB | 14 GB SSD | Miễn phí |
| `ubuntu-20.04` | Ubuntu 20.04 | 4 core | 7 GB | 14 GB SSD | Miễn phí |
| `windows-latest` | Windows Server 2022 | 4 core | 16 GB | 14 GB SSD | 2x phút |
| `windows-2022` | Windows Server 2022 | 4 core | 16 GB | 14 GB SSD | 2x phút |
| `windows-2019` | Windows Server 2019 | 4 core | 16 GB | 14 GB SSD | 2x phút |
| `macos-latest` | macOS 14 (Sonoma) | 4 core | 14 GB | 14 GB SSD | 10x phút |
| `macos-14` | macOS 14 (Sonoma) | 4 core | 14 GB | 14 GB SSD | 10x phút |
| `macos-13` | macOS 13 (Ventura) | 4 core | 14 GB | 14 GB SSD | 10x phút |

> **Billing Multiplier (Hệ Số Tính Phí):** Ubuntu = 1x, Windows = 2x, macOS = 10x

### Software Cài Sẵn

GitHub-hosted runners đi kèm **rất nhiều tool** được cài sẵn:

**Ubuntu:**
```
Docker, Node.js (LTS), Python (3.x), Java (JDK 8, 11, 17, 21),
Go, Ruby, .NET SDK, PHP, Rust, Swift,
Git, GitHub CLI (gh), AWS CLI, Azure CLI, Google Cloud CLI,
kubectl, Helm, Terraform, Ansible,
jq, curl, wget, unzip, make, gcc...
```

**Windows:**
```
Choco, Scoop, .NET Framework/SDK, Node.js, Python,
Java JDK, Go, Git, PowerShell Core,
Visual Studio Build Tools, Azure CLI, AWS CLI...
```

**macOS:**
```
Homebrew, Xcode, Node.js, Python, Java JDK,
Go, Ruby, CocoaPods, fastlane,
AWS CLI, Azure CLI...
```

Xem danh sách đầy đủ tại: [actions/runner-images](https://github.com/actions/runner-images)

### Đặc Điểm GitHub-hosted Runners

- **Ephemeral (Tạm Thời):** Mỗi job nhận runner sạch, không có data từ job trước
- **Tự động nâng cấp:** GitHub quản lý updates — không cần lo về OS patches
- **Isolated (Cô Lập):** Mỗi runner là VM riêng biệt, không share với người khác
- **Stateless:** Không có persistent storage — phải dùng cache/artifacts để giữ data

### Sử Dụng

```yaml
jobs:
  ubuntu-job:
    runs-on: ubuntu-latest

  windows-job:
    runs-on: windows-latest

  macos-job:
    runs-on: macos-latest

  # Chỉ định phiên bản cụ thể để tránh breaking changes
  ubuntu-specific:
    runs-on: ubuntu-22.04
```

---

## Self-hosted Runners

### Khi Nào Dùng Self-hosted

- Cần truy cập **mạng nội bộ** (private network — database, VPN, on-prem services)
- Cần **phần cứng đặc biệt** (GPU cho ML, ARM, thiết bị IoT)
- Yêu cầu **bảo mật cao** (code không được rời on-premise)
- Cần **tốc độ** — runner gần với deployment target
- Muốn **kiểm soát chi phí** khi có nhiều minutes cần dùng

### Cài Đặt Và Đăng Ký

```bash
# 1. Tải runner package từ GitHub (Settings → Actions → Runners)
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.317.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.317.0.tar.gz

# 2. Cấu hình runner
./config.sh \
  --url https://github.com/your-org/your-repo \
  --token YOUR_REGISTRATION_TOKEN \
  --name my-runner \
  --labels linux,self-hosted,production \
  --work _work

# 3. Chạy runner
./run.sh

# 4. (Production) Cài đặt như service systemd
sudo ./svc.sh install
sudo ./svc.sh start
```

### Sử Dụng Self-hosted Runner

```yaml
jobs:
  build:
    runs-on: self-hosted          # runner bất kỳ có label 'self-hosted'

  deploy:
    runs-on: [self-hosted, linux, production]  # phải match TẤT CẢ labels

  gpu-training:
    runs-on: [self-hosted, gpu, linux]         # runner có GPU
```

### Runner Groups — Nhóm Runner

Runner Groups (Nhóm Runner) cho phép tổ chức và kiểm soát access vào runners:

```
Organization
├── Default Group (mặc định, tất cả repos có thể dùng)
├── Production Runners Group (chỉ repos được chọn)
│   ├── prod-runner-01
│   └── prod-runner-02
└── GPU Runners Group (chỉ ML team)
    ├── gpu-runner-01
    └── gpu-runner-02
```

```yaml
# Chỉ định runner group trong workflow
jobs:
  deploy:
    runs-on:
      group: production-runners    # tên group
      labels: [linux, x64]         # filter thêm trong group
```

### Ephemeral Self-hosted Runners (Runners Tạm Thời)

Ephemeral (Tạm Thời) runners — xóa sau mỗi job, tăng bảo mật:

```bash
./config.sh \
  --url https://github.com/your-org/your-repo \
  --token YOUR_TOKEN \
  --ephemeral              # runner tự xóa sau khi xong một job
```

Thường kết hợp với auto-scaling (ARC — Actions Runner Controller) để tự tạo/xóa runners theo nhu cầu.

### Bảo Mật Self-hosted Runners

```yaml
# ⚠️ CẢNH BÁO: Không bao giờ dùng self-hosted runner cho public repos
# Attacker có thể PR code độc hại → chạy trên self-hosted runner của bạn

# ✅ Self-hosted runner chỉ an toàn khi:
# - Repository private
# - Hoặc workflow được kiểm tra kỹ trước khi chạy
# - Ephemeral runners được dùng
```

---

## So Sánh Chi Tiết

| Tiêu Chí | GitHub-hosted | Self-hosted |
|---|---|---|
| **Setup** | Không cần | Phải cài đặt và cấu hình |
| **Maintenance** | GitHub lo | Tự lo (updates, patches) |
| **Chi phí** | Trả theo phút | Trả tiền server/VM |
| **Private network** | Không | Có |
| **Custom hardware** | Không | Có |
| **Persistent cache** | Cache qua actions/cache | Filesystem tồn tại giữa jobs |
| **Startup time** | 30–60 giây | 5–15 giây |
| **Bảo mật** | Cao (isolated VM) | Tự chịu trách nhiệm |
| **Scaling** | Tự động (GitHub quản lý) | Tự implement (ARC, Terraform...) |
| **Phù hợp** | Hầu hết use cases | Private network, compliance, cost |

### Khi Nào Nên Chọn Gì

```
GitHub-hosted runner khi:
  ✓ Public repos (miễn phí)
  ✓ Standard build environments
  ✓ Không cần private network access
  ✓ Không muốn quản lý infrastructure

Self-hosted runner khi:
  ✓ Cần kết nối database, Redis, API nội bộ
  ✓ Compliance — code không ra ngoài network
  ✓ Custom hardware (GPU, ARM, IoT)
  ✓ Tốn nhiều CI minutes (>10,000 phút/tháng)
  ✓ Cần tốc độ khởi động nhanh hơn
```

---

## Larger Runners

Larger Runners (Runner Lớn Hơn) là GitHub-hosted runners với cấu hình cao hơn, yêu cầu GitHub Team hoặc Enterprise:

| Label | CPU | RAM | OS |
|---|---|---|---|
| `ubuntu-latest-4-cores` | 4 core | 16 GB | Ubuntu |
| `ubuntu-latest-8-cores` | 8 core | 32 GB | Ubuntu |
| `ubuntu-latest-16-cores` | 16 core | 64 GB | Ubuntu |
| `ubuntu-latest-32-cores` | 32 core | 128 GB | Ubuntu |
| `ubuntu-latest-64-cores` | 64 core | 256 GB | Ubuntu |
| `windows-latest-8-cores` | 8 core | 32 GB | Windows |
| `macos-latest-xlarge` | 12 core M1 | 30 GB | macOS |

```yaml
jobs:
  heavy-build:
    runs-on: ubuntu-latest-16-cores  # 16 cores cho build phức tạp
```

---

## Container Jobs

Container Jobs cho phép chạy toàn bộ job trong Docker container thay vì trực tiếp trên runner:

### container: — Chạy Job Trong Container

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine              # image Docker
      credentials:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASSWORD }}
      env:
        NODE_ENV: test
      ports:
        - 3000:3000                      # map port host:container
      volumes:
        - /tmp:/tmp                      # mount volume
        - ${{ github.workspace }}:/app  # mount workspace
      options: >-
        --cpus 2
        --memory 4g
        --user root

    steps:
      - uses: actions/checkout@v4
      - run: node --version              # chạy TRONG container node:20-alpine
      - run: npm test
```

### services: — Service Containers (Database, Cache...)

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: --health-cmd "redis-cli ping" --health-interval 10s --health-retries 5

      elasticsearch:
        image: elasticsearch:8.11.0
        env:
          discovery.type: single-node
          xpack.security.enabled: false
        ports:
          - 9200:9200

    steps:
      - uses: actions/checkout@v4

      - name: Run integration tests
        run: go test ./integration/...
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          ELASTICSEARCH_URL: http://localhost:9200
```

---

## Runner Labels

Labels (Nhãn) dùng để chỉ định runner phù hợp cho job:

### Labels Mặc Định

```yaml
# GitHub-hosted runner labels
runs-on: ubuntu-latest
runs-on: ubuntu-22.04
runs-on: windows-latest
runs-on: macos-latest

# Self-hosted labels luôn bao gồm
runs-on: self-hosted          # tự động thêm cho mọi self-hosted runner
runs-on: linux                # OS: linux, windows, macOS
runs-on: x64                  # architecture: x64, ARM, ARM64
```

### Custom Labels

```bash
# Khi đăng ký self-hosted runner
./config.sh --labels linux,production,gpu,high-memory
```

```yaml
# Sử dụng custom labels
jobs:
  ml-training:
    runs-on: [self-hosted, gpu, linux]   # phải match TẤT CẢ

  prod-deploy:
    runs-on: [self-hosted, production]
```

### Matrix Với Nhiều Runners

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

---

## Best Practices

### 1. Chỉ Định Phiên Bản Cụ Thể

```yaml
# ❌ Không ổn định — ubuntu-latest thay đổi theo thời gian
runs-on: ubuntu-latest

# ✅ Ổn định hơn — biết chính xác OS đang dùng
runs-on: ubuntu-22.04
```

### 2. Dùng Ephemeral Self-hosted Runners

```bash
# Ephemeral — không có state leakage giữa các jobs
./config.sh --ephemeral
```

### 3. Không Dùng Self-hosted Cho Public Repos

```yaml
# ⚠️ Nguy hiểm với public repo — ai cũng có thể PR code chạy trên self-hosted của bạn
on:
  pull_request:
jobs:
  build:
    runs-on: self-hosted  # ❌ Rủi ro bảo mật cực cao

# ✅ Dùng GitHub-hosted cho public repos
    runs-on: ubuntu-latest
```

### 4. Tách Runner Môi Trường

```
Production runners: chỉ deploy scripts, không build
Build runners: chỉ compile code, không có prod credentials
```

### 5. Health Check Cho Service Containers

```yaml
services:
  db:
    image: postgres:15
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5       # chờ service sẵn sàng trước khi bắt đầu job
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Khi nào nên dùng self-hosted runner thay vì GitHub-hosted?**

A: Ba lý do chính:
1. **Private network access** — cần kết nối tới database, API, hoặc services nội bộ không public
2. **Compliance/Security** — yêu cầu code và credentials không rời môi trường nội bộ (on-premise, air-gapped)
3. **Cost optimization** — khi sử dụng nhiều minutes và chi phí self-hosted server rẻ hơn

**Q: `container:` và `services:` khác nhau như thế nào?**

A:
- `container:` — Toàn bộ job chạy TRONG container đó. Các steps dùng shell của container.
- `services:` — Container chạy song song với job (không phải trong đó), expose ports để job kết nối vào. Dùng cho dependencies như database, cache.

**Q: Self-hosted runner có persistent storage không?**

A: Có! Khác với GitHub-hosted (ephemeral), self-hosted runner giữ filesystem giữa các jobs. Điều này có thể là lợi thế (cache nhanh hơn) nhưng cũng là rủi ro bảo mật (state từ job trước có thể ảnh hưởng job sau). Dùng `--ephemeral` flag để tạo runner tạm thời nếu cần behavior giống GitHub-hosted.

---

## 📂 Điều Hướng

- [← Events & Triggers](2-events-triggers.md)
- [→ Contexts & Expressions](4-contexts-expressions.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
