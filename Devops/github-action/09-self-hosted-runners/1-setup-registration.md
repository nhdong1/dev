# Cài Đặt & Đăng Ký Self-hosted Runner

> Hướng dẫn từng bước cài đặt runner binary, đăng ký với GitHub, cấu hình labels (nhãn), runner groups (nhóm runner), và chạy như systemd service trên môi trường Linux.

---

## 📚 Mục Lục

1. [Yêu Cầu Hệ Thống](#yêu-cầu-hệ-thống)
2. [Cài Đặt Runner Binary](#cài-đặt-runner-binary)
3. [Đăng Ký Runner](#đăng-ký-runner)
4. [Labels và Runner Groups](#labels-và-runner-groups)
5. [Chạy Runner Như Service](#chạy-runner-như-service)
6. [Sử Dụng Trong Workflow](#sử-dụng-trong-workflow)
7. [Đăng Ký Runner Cho Organization và Enterprise](#đăng-ký-runner-cho-organization-và-enterprise)
8. [Bài Tập Thực Hành](#bài-tập-thực-hành)

---

## 🖥️ Yêu Cầu Hệ Thống

### Hệ Điều Hành Được Hỗ Trợ

| OS | Kiến Trúc | Ghi Chú |
|---|---|---|
| Ubuntu 20.04 / 22.04 / 24.04 | x64, ARM64 | Khuyến nghị |
| Debian 10+ | x64, ARM64 | Hỗ trợ đầy đủ |
| Red Hat Enterprise Linux 8/9 | x64 | Hỗ trợ đầy đủ |
| Windows Server 2019/2022 | x64 | Hỗ trợ đầy đủ |
| macOS 12+ (Monterey trở lên) | x64, ARM64 | Hỗ trợ đầy đủ |

### Yêu Cầu Tối Thiểu

- **CPU:** 2 vCPU
- **RAM:** 4 GB (khuyến nghị 8 GB cho build nặng)
- **Disk:** 14 GB (khuyến nghị 50 GB để cache Docker layers, Maven repo, npm packages)
- **Network:** Kết nối HTTPS outbound đến `github.com`, `api.github.com`, `*.actions.githubusercontent.com`
- **Software:** `curl`, `tar`, `git` (phiên bản 2.18+)

### Danh Sách Domain Cần Cho Phép Outbound

```
# Thêm vào whitelist firewall / proxy
github.com
api.github.com
codeload.github.com
*.actions.githubusercontent.com
objects.githubusercontent.com
pkg.github.com
ghcr.io
*.pkg.github.com
npm.pkg.github.com
pipelines.actions.githubusercontent.com
results-receiver.actions.githubusercontent.com
```

---

## 📦 Cài Đặt Runner Binary

### Bước 1: Tạo User Chuyên Dụng (Khuyến Nghị)

```bash
# Tạo user 'runner' không có home directory và không có login shell
sudo useradd --system --no-create-home --shell /bin/false runner

# Tạo thư mục cài đặt
sudo mkdir -p /opt/actions-runner
sudo chown runner:runner /opt/actions-runner

# Chuyển sang user runner từ đây
sudo -u runner -s /bin/bash
cd /opt/actions-runner
```

> **Lý do:** Chạy runner dưới non-root user hạn chế blast radius khi có code độc hại chạy trong workflow.

### Bước 2: Tải Runner Binary

```bash
# Lấy phiên bản mới nhất từ GitHub API
RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest \
  | grep '"tag_name"' | sed -E 's/.*"v([^"]+)".*/\1/')

echo "Phiên bản runner mới nhất: ${RUNNER_VERSION}"

# Tải binary (Linux x64)
curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

# Xác minh SHA-256 (bước quan trọng về bảo mật)
# Lấy expected hash từ GitHub release page
echo "<expected_sha256_hash>  actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" | shasum -a 256 -c

# Giải nén
tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
```

### Cài Đặt Trên ARM64

```bash
# Thay x64 bằng arm64 cho máy chủ ARM (ví dụ: AWS Graviton, Raspberry Pi)
curl -o actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz -L \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz"
```

---

## 🔑 Đăng Ký Runner

### Lấy Registration Token (Token Đăng Ký)

Registration token có hiệu lực trong 1 giờ. Lấy qua GitHub UI hoặc API.

#### Qua GitHub UI

```
Repository:   Settings → Actions → Runners → New self-hosted runner
Organization: Settings → Actions → Runners → New runner → New self-hosted runner
Enterprise:   Settings → Actions → Runner groups → New runner
```

#### Qua GitHub API

```bash
# Lấy token cho repository
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/{owner}/{repo}/actions/runners/registration-token"

# Response:
# {
#   "token": "ABCDEFGHIJKLMNOP...",
#   "expires_at": "2026-05-13T00:00:00.000-00:00"
# }
```

### Chạy Config Script

```bash
cd /opt/actions-runner

# Cấu hình runner
./config.sh \
  --url https://github.com/{owner}/{repo} \
  --token <REGISTRATION_TOKEN> \
  --name "prod-runner-01" \
  --labels "self-hosted,linux,x64,production,docker" \
  --runnergroup "Production" \
  --work "_work" \
  --replace    # Ghi đè nếu runner cùng tên đã tồn tại
```

**Giải thích các flags:**

| Flag | Ý Nghĩa |
|---|---|
| `--url` | URL của repo, org, hoặc enterprise |
| `--token` | Registration token (một lần dùng) |
| `--name` | Tên hiển thị trong GitHub UI |
| `--labels` | Danh sách labels phân cách bằng dấu phẩy |
| `--runnergroup` | Nhóm runner (chỉ có ở Organization/Enterprise) |
| `--work` | Thư mục làm việc cho jobs |
| `--replace` | Thay thế runner cùng tên nếu tồn tại |
| `--ephemeral` | Runner xóa sau mỗi job (bảo mật cao hơn) |

### Chế Độ Ephemeral (Khuyến Nghị Cho Production)

```bash
./config.sh \
  --url https://github.com/{owner}/{repo} \
  --token <REGISTRATION_TOKEN> \
  --name "ephemeral-runner" \
  --labels "self-hosted,linux,ephemeral" \
  --ephemeral    # Runner tự xóa registration sau khi hoàn thành 1 job
```

> **Ephemeral runners** (Runners Tạm Thời) — mỗi job chạy trên môi trường sạch, không có state kéo theo từ job trước. Đây là best practice bảo mật tương đương GitHub-hosted runners.

---

## 🏷️ Labels và Runner Groups

### Labels (Nhãn) — Cách Hoạt Động

Labels là cơ chế routing để workflow chọn đúng runner. Một runner có thể có nhiều labels, workflow dùng `runs-on` để match.

```yaml
# Workflow chỉ chạy trên runner có đủ CẢ HAI labels: self-hosted VÀ gpu
jobs:
  train-model:
    runs-on: [self-hosted, gpu]
```

GitHub match runner nào có **tất cả** labels được chỉ định.

### Quy Ước Đặt Labels

```
# Label chuẩn GitHub (luôn thêm)
self-hosted       # Phân biệt với GitHub-hosted
linux             # OS
x64               # Kiến trúc (hoặc arm64)

# Labels theo môi trường
production        # Runner production (access DB prod)
staging           # Runner staging
build             # Runner chuyên build

# Labels theo phần cứng
high-memory       # 32GB+ RAM
gpu               # Có GPU
docker            # Docker daemon sẵn sàng
k8s-node          # Chạy trên Kubernetes node

# Labels theo team
team-backend      # Chỉ team backend dùng
team-data         # Team data science
```

### Runner Groups (Nhóm Runner) — Organization/Enterprise

Runner Groups kiểm soát **ai được dùng** runner nào. Chỉ có ở Organization và Enterprise plan.

```
Organization Settings → Actions → Runner groups

Mặc định:
- "Default" group: tất cả repositories trong org có thể dùng

Tạo group riêng:
- "Production" group: chỉ repo "app-api" và "app-frontend" được phép
- "GPU" group: chỉ repo "ml-training" được phép
```

**Tạo Runner Group qua API:**

```bash
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/orgs/{org}/actions/runner-groups" \
  -d '{
    "name": "Production",
    "visibility": "selected",
    "selected_repository_ids": [123456, 789012],
    "allows_public_repositories": false,
    "restricted_to_workflows": true,
    "selected_workflows": ["deploy-production.yml"]
  }'
```

> `restricted_to_workflows` — chỉ cho phép workflow cụ thể dùng runner group này, tăng cường bảo mật.

---

## ⚙️ Chạy Runner Như Service

### Systemd Service (Linux)

```bash
# Cài đặt service (chạy với sudo, nhưng service sẽ chạy dưới user 'runner')
sudo /opt/actions-runner/svc.sh install runner

# Khởi động service
sudo systemctl start actions.runner.<owner>.<repo>.<name>.service

# Bật tự khởi động cùng hệ thống
sudo systemctl enable actions.runner.<owner>.<repo>.<name>.service

# Kiểm tra trạng thái
sudo systemctl status actions.runner.<owner>.<repo>.<name>.service
```

### Tạo Systemd Unit File Thủ Công (Tùy Chỉnh Hơn)

```ini
# /etc/systemd/system/github-runner.service
[Unit]
Description=GitHub Actions Runner
After=network.target

[Service]
ExecStart=/opt/actions-runner/run.sh
User=runner
Group=runner
WorkingDirectory=/opt/actions-runner
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=5min
Restart=always
RestartSec=10

# Giới hạn tài nguyên (Resource Limits — Giới Hạn Tài Nguyên)
LimitNOFILE=65536
LimitNPROC=512

# Bảo mật bổ sung
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/actions-runner

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable github-runner
sudo systemctl start github-runner
```

### Windows Service

```powershell
# Chạy trong PowerShell với quyền Administrator
cd C:\actions-runner
.\config.cmd --url https://github.com/{owner}/{repo} --token <TOKEN>

# Cài đặt service
.\svc.cmd install

# Khởi động service
.\svc.cmd start

# Kiểm tra trạng thái
.\svc.cmd status
```

---

## 🔧 Sử Dụng Trong Workflow

### Targeting Runner Cơ Bản

```yaml
name: Build on Self-hosted
on: push

jobs:
  build:
    # Chỉ chạy trên runner có label 'self-hosted' và 'linux'
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on self-hosted runner"
```

### Targeting Theo Môi Trường

```yaml
jobs:
  deploy-staging:
    runs-on: [self-hosted, linux, staging]
    environment: staging
    steps:
      - name: Deploy to staging
        run: ./deploy.sh staging

  deploy-production:
    runs-on: [self-hosted, linux, production]
    environment: production
    needs: [deploy-staging]
    steps:
      - name: Deploy to production
        run: ./deploy.sh production
```

### Sử Dụng Runner Name Cụ Thể

```yaml
jobs:
  gpu-training:
    # Chạy trên runner có GPU
    runs-on:
      group: GPU          # Runner group (chỉ ở Organization)
      labels: [self-hosted, gpu, linux]
    steps:
      - name: Train model
        run: python train.py --gpu
```

### Kết Hợp GitHub-hosted và Self-hosted Trong Matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        include:
          - os: ubuntu-latest         # GitHub-hosted
            runner: ubuntu-latest
          - os: ubuntu-22.04-arm64    # Self-hosted ARM
            runner: [self-hosted, linux, arm64]
          - os: windows-2022          # GitHub-hosted Windows
            runner: windows-latest
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - run: echo "Testing on ${{ matrix.os }}"
```

---

## 🏢 Đăng Ký Runner Cho Organization và Enterprise

### Organization-level Runner

Runner cấp organization có thể được dùng bởi nhiều repositories trong org.

```bash
# Lấy token đăng ký org runner
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/orgs/{org}/actions/runners/registration-token"

# Đăng ký runner vào org
./config.sh \
  --url https://github.com/{org} \    # URL là org, không phải repo
  --token <ORG_REGISTRATION_TOKEN> \
  --name "org-runner-01" \
  --labels "self-hosted,linux,org-shared" \
  --runnergroup "Default"
```

### Enterprise-level Runner

```bash
# Lấy token đăng ký enterprise runner
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/enterprises/{enterprise}/actions/runners/registration-token"

# Đăng ký runner vào enterprise
./config.sh \
  --url https://github.com/enterprises/{enterprise} \
  --token <ENTERPRISE_REGISTRATION_TOKEN> \
  --name "enterprise-runner-01" \
  --labels "self-hosted,linux,enterprise" \
  --runnergroup "Enterprise-Shared"
```

### Phạm Vi Runner (Runner Scope)

```
Enterprise Runner
  ├── Dùng được bởi tất cả organizations trong enterprise
  └── Quản lý tại: Enterprise Settings → Actions → Runners

Organization Runner
  ├── Dùng được bởi tất cả repos trong org (hoặc repos được chọn)
  └── Quản lý tại: Org Settings → Actions → Runners

Repository Runner
  ├── Chỉ dùng được bởi repo đó
  └── Quản lý tại: Repo Settings → Actions → Runners
```

---

## 💡 Automation — Tự Động Đăng Ký Runner

### Script Đăng Ký Tự Động (Dành Cho Infrastructure As Code)

```bash
#!/bin/bash
# auto-register-runner.sh
# Dùng để khởi tạo runner trên VM mới (cloud-init, Ansible, Terraform userdata)

set -euo pipefail

GITHUB_OWNER="${GITHUB_OWNER}"          # export trước khi chạy
GITHUB_REPO="${GITHUB_REPO}"
GITHUB_PAT="${GITHUB_PAT}"              # PAT với scope: repo, admin:org
RUNNER_NAME="${RUNNER_NAME:-$(hostname)}"
RUNNER_LABELS="${RUNNER_LABELS:-self-hosted,linux,x64}"
RUNNER_GROUP="${RUNNER_GROUP:-Default}"
RUNNER_INSTALL_DIR="/opt/actions-runner"

# Lấy registration token
REG_TOKEN=$(curl -s -X POST \
  -H "Authorization: token ${GITHUB_PAT}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/actions/runners/registration-token" \
  | jq -r '.token')

if [[ -z "$REG_TOKEN" || "$REG_TOKEN" == "null" ]]; then
  echo "ERROR: Không lấy được registration token" >&2
  exit 1
fi

# Lấy phiên bản mới nhất
RUNNER_VERSION=$(curl -s \
  https://api.github.com/repos/actions/runner/releases/latest \
  | jq -r '.tag_name' | sed 's/v//')

# Tải và cài đặt
mkdir -p "${RUNNER_INSTALL_DIR}"
cd "${RUNNER_INSTALL_DIR}"

curl -sL -o runner.tar.gz \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"
tar xzf runner.tar.gz
rm runner.tar.gz

# Đăng ký
./config.sh \
  --url "https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}" \
  --token "${REG_TOKEN}" \
  --name "${RUNNER_NAME}" \
  --labels "${RUNNER_LABELS}" \
  --runnergroup "${RUNNER_GROUP}" \
  --unattended \    # Không hỏi input
  --ephemeral       # Xóa sau mỗi job

# Cài service
./svc.sh install
systemctl start "actions.runner.${GITHUB_OWNER}.${GITHUB_REPO}.${RUNNER_NAME}.service"

echo "Runner ${RUNNER_NAME} đã đăng ký thành công"
```

### Terraform Integration

```hcl
# main.tf — Tự động tạo runner VM trên AWS và đăng ký

resource "aws_instance" "github_runner" {
  count         = var.runner_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  user_data = templatefile("${path.module}/runner-init.sh.tpl", {
    github_owner   = var.github_owner
    github_repo    = var.github_repo
    runner_labels  = "self-hosted,linux,x64,aws"
    runner_group   = "AWS-Runners"
  })

  tags = {
    Name = "github-runner-${count.index + 1}"
  }
}
```

---

## 🧪 Bài Tập Thực Hành

### Bài 1: Cài Đặt Runner Trên Ubuntu VM

```
Mục tiêu: Cài đặt và đăng ký self-hosted runner cho repository thực hành

1. Tạo VM Ubuntu 22.04 (có thể dùng VirtualBox, Multipass, hoặc cloud VM)
2. Tạo user 'runner' chuyên dụng
3. Cài đặt runner binary
4. Đăng ký runner với repository GitHub của bạn
5. Thêm labels: self-hosted,linux,x64,practice
6. Cài đặt systemd service và enable
7. Xác nhận runner xuất hiện trong GitHub Settings → Actions → Runners
```

### Bài 2: Workflow Targeting Self-hosted Runner

```yaml
# .github/workflows/self-hosted-test.yml
name: Test Self-hosted Runner
on:
  workflow_dispatch:    # Kích hoạt thủ công

jobs:
  test:
    runs-on: [self-hosted, linux, practice]
    steps:
      - name: System info
        run: |
          echo "Hostname: $(hostname)"
          echo "User: $(whoami)"
          echo "OS: $(uname -a)"
          echo "CPU: $(nproc) cores"
          echo "RAM: $(free -h | grep Mem | awk '{print $2}')"

      - name: Verify not root
        run: |
          if [ "$(id -u)" = "0" ]; then
            echo "FAIL: Runner đang chạy dưới root — rủi ro bảo mật!"
            exit 1
          fi
          echo "OK: Runner chạy dưới user $(whoami)"
```

### Bài 3: So Sánh GitHub-hosted vs Self-hosted Build Time

```yaml
# .github/workflows/build-comparison.yml
name: Build Time Comparison
on: workflow_dispatch

jobs:
  github-hosted:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: |
          time npm ci
          time npm run build

  self-hosted:
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4
      - name: Build (with local cache)
        run: |
          time npm ci        # Cache node_modules trên disk
          time npm run build
```

---

## 📋 Tóm Tắt Nhanh

| Bước | Lệnh / Hành Động |
|---|---|
| Tạo user | `sudo useradd --system runner` |
| Tải binary | `curl -o runner.tar.gz <URL>` |
| Đăng ký | `./config.sh --url ... --token ...` |
| Ephemeral | Thêm `--ephemeral` vào config.sh |
| Cài service | `sudo ./svc.sh install` |
| Khởi động | `sudo systemctl start <service>` |
| Kiểm tra | `sudo systemctl status <service>` |
| Trong workflow | `runs-on: [self-hosted, linux]` |

---

**Tiếp Theo:** [2-arc-autoscaling.md](./2-arc-autoscaling.md) — ARC và auto-scaling trên Kubernetes
