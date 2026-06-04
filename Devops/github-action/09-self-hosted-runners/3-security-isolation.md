# Bảo Mật & Cô Lập Self-hosted Runners

> Self-hosted runners mang lại sức mạnh lớn nhưng cũng đi kèm rủi ro bảo mật nghiêm trọng nếu không được cấu hình đúng. File này trình bày các mối đe dọa, chiến lược cô lập mạng, ephemeral runners, và checklist hardening (tăng cường bảo mật) toàn diện.

---

## 📚 Mục Lục

1. [Mô Hình Đe Dọa](#mô-hình-đe-dọa)
2. [Cô Lập Mạng](#cô-lập-mạng)
3. [Ephemeral Runners](#ephemeral-runners)
4. [Hardening Runner Environment](#hardening-runner-environment)
5. [Bảo Vệ Secrets Trong Runner](#bảo-vệ-secrets-trong-runner)
6. [Runner Groups và Access Control](#runner-groups-và-access-control)
7. [Kiểm Soát Workflow Được Phép Chạy](#kiểm-soát-workflow-được-phép-chạy)
8. [Audit và Compliance](#audit-và-compliance)
9. [Checklist Bảo Mật Toàn Diện](#checklist-bảo-mật-toàn-diện)

---

## ⚠️ Mô Hình Đe Dọa

### Rủi Ro Lớn Nhất: Code Injection Qua Pull Request

```
Kịch bản tấn công:
1. Attacker fork repository public của bạn
2. Tạo PR với workflow thay đổi để chạy lệnh độc hại
3. Workflow chạy trên self-hosted runner của bạn
4. Attacker có thể đọc file trên runner, steal credentials từ environment,
   truy cập mạng nội bộ, hoặc persist malware trên runner
```

**Đây là lý do tại sao GitHub khuyến cáo mạnh: KHÔNG BAO GIỜ dùng self-hosted runners cho public repositories.**

### Các Vector Tấn Công Phổ Biến

| Vector | Mô Tả | Mức Độ |
|---|---|---|
| **Code injection qua PR** | Malicious code trong workflow của PR | Nghiêm trọng |
| **Credentials theft** | Đọc env vars, files chứa secrets | Nghiêm trọng |
| **Lateral movement** | Từ runner tấn công sang các services nội bộ | Nghiêm trọng |
| **Persistent backdoor** | Cài malware tồn tại sau khi job kết thúc | Cao |
| **Supply chain attack** | Action độc hại từ Marketplace | Cao |
| **Cache poisoning** | Đầu độc cache để ảnh hưởng build sau | Trung bình |

### Runners Cho Public vs Private Repository

```
Public Repository:
  ❌ Self-hosted runners — KHÔNG BAO GIỜ dùng
  ✅ GitHub-hosted runners — An toàn, môi trường sạch mỗi lần

Private Repository:
  ✅ Self-hosted runners — Được dùng, nhưng phải hardening
  ✅ GitHub-hosted runners — Vẫn là lựa chọn tốt hơn nếu không cần đặc thù
```

---

## 🔒 Cô Lập Mạng

### Nguyên Tắc Least-privilege Mạng

Runner chỉ cần kết nối ra ngoài một chiều (outbound) đến GitHub. Mọi kết nối khác phải được kiểm soát chặt chẽ.

```
Cho phép (Whitelist):
  Outbound HTTPS (443) → github.com
  Outbound HTTPS (443) → api.github.com
  Outbound HTTPS (443) → *.actions.githubusercontent.com
  Outbound HTTPS (443) → *.pkg.github.com

Chặn:
  Outbound → Mọi internet khác (trừ những gì cần thiết cho build)
  Inbound  → Tất cả (runner không cần nhận kết nối vào)
  Lateral  → Chặn runner kết nối sang segments mạng khác
```

### AWS VPC Security Group

```hcl
# Terraform — Security Group cho runner EC2 instances
resource "aws_security_group" "github_runner" {
  name        = "github-runner"
  description = "Self-hosted GitHub Actions runners"
  vpc_id      = aws_vpc.main.id

  # Không có inbound rules — runner không nhận kết nối vào

  # Outbound — chỉ HTTPS
  egress {
    description = "HTTPS to GitHub"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Nếu cần kết nối đến private resources
  egress {
    description     = "Database access"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]
  }

  tags = { Name = "github-runner-sg" }
}
```

### Kubernetes NetworkPolicy (Chính Sách Mạng Kubernetes)

```yaml
# Cô lập network cho runner Pods trên ARC
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: arc-runner-isolation
  namespace: arc-runners
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: runner

  policyTypes:
    - Ingress
    - Egress

  ingress: []    # Chặn TẤT CẢ inbound — runner không nhận kết nối

  egress:
    # GitHub Actions API
    - ports:
        - port: 443
          protocol: TCP

    # DNS resolution (Phân Giải Tên Miền)
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # Private registry (nếu pull images nội bộ)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: registry
      ports:
        - port: 5000
          protocol: TCP
```

### Egress Proxy (Proxy Đầu Ra)

Với môi trường enterprise cần kiểm soát outbound traffic:

```bash
# Cấu hình runner dùng corporate proxy
export HTTPS_PROXY="https://proxy.internal.corp:8080"
export NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,*.internal.corp"

# Trong systemd service
[Service]
Environment=HTTPS_PROXY=https://proxy.internal.corp:8080
Environment=NO_PROXY=localhost,127.0.0.1,10.0.0.0/8
```

---

## 🔄 Ephemeral Runners

### Tại Sao Ephemeral?

```
Non-ephemeral runner (Không tạm thời):
  Job 1 (attacker) → cài backdoor vào ~/.bashrc hoặc /tmp
  Job 2 (legitimate) → chạy trên runner đã bị compromise
  → Tấn công persistence, data exfiltration

Ephemeral runner (Tạm thời):
  Job 1 (attacker) → làm gì tùy thích
  Runner xóa → môi trường sạch hoàn toàn
  Job 2 (legitimate) → chạy trên môi trường sạch
  → Tấn công không persist
```

### Cấu Hình Ephemeral Trên VM

```bash
# --ephemeral flag: runner tự hủy đăng ký sau 1 job
./config.sh \
  --url https://github.com/{owner}/{repo} \
  --token <TOKEN> \
  --ephemeral

# Kết hợp với systemd để tự tạo lại runner sau mỗi job
```

**Systemd service cho ephemeral VM runner:**

```ini
# /etc/systemd/system/ephemeral-runner.service
[Unit]
Description=Ephemeral GitHub Actions Runner
After=network.target

[Service]
Type=simple
User=runner
WorkingDirectory=/opt/actions-runner

# Script chạy một vòng lặp: config → run → unregister → config lại
ExecStart=/opt/actions-runner/run-ephemeral.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
#!/bin/bash
# /opt/actions-runner/run-ephemeral.sh
# Vòng lặp ephemeral runner: mỗi lần chạy = 1 job

while true; do
  # Lấy token mới cho mỗi vòng
  REG_TOKEN=$(curl -s -X POST \
    -H "Authorization: token ${GITHUB_PAT}" \
    "https://api.github.com/repos/${OWNER}/${REPO}/actions/runners/registration-token" \
    | jq -r '.token')

  # Cấu hình runner
  ./config.sh \
    --url "https://github.com/${OWNER}/${REPO}" \
    --token "${REG_TOKEN}" \
    --name "ephemeral-$(hostname)-$(date +%s)" \
    --ephemeral \
    --unattended \
    --replace

  # Chạy runner (block cho đến khi job kết thúc)
  ./run.sh

  # Dọn dẹp workspace sau job
  rm -rf _work/*

  echo "Job hoàn thành. Đang khởi tạo lại runner..."
  sleep 2
done
```

### Ephemeral Containers Với Docker

```bash
# Chạy runner như Docker container — tự xóa sau khi xong
docker run --rm \
  -e RUNNER_NAME="ephemeral-$(hostname)" \
  -e GITHUB_URL="https://github.com/{owner}/{repo}" \
  -e RUNNER_TOKEN="$(get_token)" \
  -e RUNNER_LABELS="self-hosted,linux,docker,ephemeral" \
  ghcr.io/actions/actions-runner:latest
```

```yaml
# Docker Compose cho fleet ephemeral runners
# docker-compose.yml
version: '3.8'

services:
  runner:
    image: ghcr.io/actions/actions-runner:latest
    deploy:
      replicas: 5    # Số runners chờ sẵn
    environment:
      - GITHUB_URL=https://github.com/{owner}/{repo}
      - RUNNER_LABELS=self-hosted,linux,ephemeral
    env_file:
      - runner.env    # Chứa RUNNER_TOKEN

    # Không mount host filesystem (trừ Docker socket nếu cần)
    volumes: []

    # Restart tạo container mới (ephemeral)
    restart: unless-stopped
```

---

## 🛡️ Hardening Runner Environment

### Systemd Service Hardening

```ini
# /etc/systemd/system/github-runner-hardened.service
[Service]
User=runner
Group=runner

# Filesystem restrictions
ProtectSystem=strict          # Hệ thống file chỉ đọc (trừ các path được chỉ định)
ProtectHome=true              # Không truy cập /home của users khác
PrivateTmp=true               # /tmp riêng biệt, không chia sẻ với processes khác
ReadWritePaths=/opt/actions-runner    # Chỉ thư mục này được ghi

# Process restrictions
NoNewPrivileges=true          # Không được leo thang đặc quyền (privilege escalation)
PrivateDevices=true           # Không truy cập /dev (trừ null, zero, tty)
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX    # Chỉ mạng cơ bản

# System call filtering (Lọc System Call — Lọc Lời Gọi Hệ Thống)
SystemCallFilter=@system-service    # Chỉ cho phép syscalls cần thiết
SystemCallArchitectures=native

# Capability restrictions (Hạn Chế Khả Năng)
CapabilityBoundingSet=        # Xóa TẤT CẢ Linux capabilities
AmbientCapabilities=          # Không có ambient capabilities
```

### Linux Security Modules

#### AppArmor Profile

```
# /etc/apparmor.d/github-runner
profile github-runner /opt/actions-runner/run.sh {
  # Cho phép đọc cơ bản
  /opt/actions-runner/** r,

  # Cho phép ghi vào workspace
  /opt/actions-runner/_work/** rw,
  /tmp/** rw,

  # Mạng
  network inet tcp,
  network inet6 tcp,

  # Chặn đọc file nhạy cảm
  deny /etc/shadow r,
  deny /etc/sudoers r,
  deny /root/** r,
  deny /home/*/.ssh/** r,
}
```

```bash
# Load AppArmor profile
sudo apparmor_parser -r /etc/apparmor.d/github-runner
```

### Disk Quota và Resource Limits

```bash
# Cấu hình disk quota cho user runner
sudo apt-get install -y quota
sudo setquota -u runner 20G 25G 0 0 /dev/sda1

# Cgroup limits — giới hạn CPU và Memory cho runner processes
sudo cgcreate -g memory,cpu:github-runner
sudo cgset -r memory.max_usage_in_bytes=4G memory github-runner
sudo cgset -r cpu.cfs_quota_us=200000 cpu github-runner    # 2 cores
```

### Docker Security Cho Container-based Runners

```yaml
# docker-compose.yml với security hardening
services:
  runner:
    image: ghcr.io/actions/actions-runner:latest
    user: "1001:1001"    # Non-root user

    security_opt:
      - no-new-privileges:true     # Không leo thang đặc quyền
      - apparmor:docker-default    # AppArmor profile

    cap_drop:
      - ALL    # Xóa tất cả Linux capabilities

    cap_add:
      - NET_BIND_SERVICE    # Chỉ thêm lại những gì thực sự cần

    read_only: true    # Container filesystem chỉ đọc

    tmpfs:
      - /tmp:noexec,nodev,size=1G    # /tmp riêng, không thể exec

    volumes:
      - runner-work:/home/runner/_work    # Workspace riêng biệt

    # Giới hạn tài nguyên
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
```

---

## 🔑 Bảo Vệ Secrets Trong Runner

### Nguyên Tắc

1. **Không bao giờ** hard-code secrets trong runner config hoặc script
2. Dùng GitHub Secrets để inject vào workflow dưới dạng env vars
3. Runner environment variables **không được persist** sau job (nếu ephemeral)
4. Giám sát logs để phát hiện secrets bị in ra

### GitHub Secrets Masking

GitHub Actions tự động mask (che giấu) giá trị của secrets trong logs.

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, linux]
    steps:
      - name: Use secret (sẽ bị mask trong logs)
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "API_KEY is: $API_KEY"
          # Logs sẽ hiển thị: API_KEY is: ***
```

### Giới Hạn Secret Access Theo Environment

```yaml
# Dùng environments để giới hạn secrets
jobs:
  deploy-prod:
    runs-on: [self-hosted, linux, production]
    environment: production    # Chỉ secrets của env 'production' được inject
    steps:
      - run: deploy.sh
        env:
          DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
```

### Không Dùng Runner-level Environment Variables Cho Secrets

```bash
# BAD: Set secret trong runner service environment
# /etc/systemd/system/github-runner.service
[Service]
Environment=AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE    # ❌ Mọi job đều thấy

# GOOD: Inject qua GitHub Secrets trong workflow
# .github/workflows/deploy.yml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}  # ✅ Chỉ workflow này thấy
```

---

## 👥 Runner Groups và Access Control

### Cấu Hình Runner Group Với Workflow Restrictions

```bash
# Tạo group chỉ cho phép workflow cụ thể
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/orgs/{org}/actions/runner-groups" \
  -d '{
    "name": "Production-Runners",
    "visibility": "selected",
    "selected_repository_ids": [123456],
    "restricted_to_workflows": true,
    "selected_workflows": [
      "org/repo/.github/workflows/deploy-production.yml@refs/heads/main"
    ]
  }'
```

### Phân Cấp Access Control

```
Enterprise Admin
  └── Quản lý Enterprise Runner Groups
      └── Chỉ định Organizations được dùng

Organization Admin
  └── Quản lý Org Runner Groups
      ├── Chọn Repositories được dùng runner
      └── Restrict to specific workflows

Repository Admin
  └── Quản lý Repo Runner
      └── Chỉ người có Admin access mới thêm/xóa runner
```

---

## 📋 Kiểm Soát Workflow Được Phép Chạy

### Pull Request Fork Policy (Chính Sách Fork PR)

```
Repository Settings → Actions → General → Fork pull request workflows

Tùy chọn:
1. "Require approval for all outside collaborators"
   → Mọi PR từ người chưa là collaborator cần approval
   → KHUYẾN NGHỊ khi có self-hosted runners

2. "Require approval for first-time contributors"
   → Chỉ cần approve lần đầu
   → RỦI RO nếu account bị compromise sau lần đầu

3. "Allow all actions and reusable workflows"
   → KHÔNG BAO GIỜ dùng với self-hosted runners
```

### Kiểm Soát Qua `pull_request_target`

```yaml
# Workflow KHÔNG chạy code từ fork branch — an toàn hơn
on:
  pull_request_target:    # Chạy code từ BASE branch, không phải HEAD của PR
    branches: [main]

jobs:
  check-pr:
    runs-on: [self-hosted, linux]
    steps:
      # CẢNH BÁO: KHÔNG checkout PR code nếu dùng pull_request_target
      # trên self-hosted runner — đây là lỗ hổng bảo mật phổ biến
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.base.sha }}    # Checkout BASE, không HEAD
```

> **Anti-pattern nguy hiểm:** Dùng `pull_request_target` kết hợp `actions/checkout@v4` với default ref (HEAD của PR) trên self-hosted runner — attacker có thể inject code độc hại.

### Workflow Concurrency Để Tránh Race Conditions

```yaml
# Giới hạn chỉ 1 deployment production chạy cùng lúc
jobs:
  deploy:
    runs-on: [self-hosted, linux, production]
    concurrency:
      group: production-deployment
      cancel-in-progress: false    # Không hủy, xếp hàng chờ
```

---

## 📊 Audit và Compliance

### GitHub Audit Log

Tất cả hoạt động runner được ghi vào Audit Log (Nhật Ký Kiểm Toán).

```bash
# Lấy audit log qua API
curl -H "Authorization: token $GITHUB_PAT" \
  "https://api.github.com/orgs/{org}/audit-log?phrase=runner&include=all&per_page=100"

# Events quan trọng cần monitor:
# runners.create       — Runner mới được đăng ký
# runners.delete       — Runner bị xóa
# runners.force_cancel — Job bị force cancel
# workflow_run.started — Workflow bắt đầu
# workflow_run.completed — Workflow kết thúc
```

### Runner Activity Logging

```bash
# Logs của runner (Linux systemd)
journalctl -u actions.runner.* --since "1 hour ago" -f

# Forward logs đến centralized system (Splunk, ELK, CloudWatch)
# /etc/systemd/journald.conf
[Journal]
Storage=persistent
ForwardToSyslog=yes

# Rsyslog forward đến ELK
# /etc/rsyslog.d/github-runner.conf
:programname, isequal, "actions.runner" @@elk-server:5514
```

### Compliance Checklist Cho SOC2 / ISO27001

```
Access Management (Quản Lý Truy Cập):
  ✅ Runner chạy dưới dedicated non-privileged user
  ✅ Runner groups restrict access to specific repositories
  ✅ Workflow restrictions giới hạn workflows được phép dùng runner
  ✅ Audit logs enabled và retained 90+ ngày

Network Security (Bảo Mật Mạng):
  ✅ Runners ở network segment riêng biệt
  ✅ Egress chỉ cho phép GitHub và services cần thiết
  ✅ Không có inbound connections
  ✅ TLS inspection nếu dùng egress proxy

Data Protection (Bảo Vệ Dữ Liệu):
  ✅ Ephemeral runners — không lưu data giữa jobs
  ✅ Workspace cleanup sau mỗi job
  ✅ Secrets không được log hoặc persist
  ✅ Disk encryption trên runner machines
```

---

## ✅ Checklist Bảo Mật Toàn Diện

### Cấu Hình Runner

- [ ] Runner chạy dưới non-root user chuyên dụng
- [ ] Systemd hardening — `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=strict`
- [ ] Ephemeral mode được bật (`--ephemeral` flag hoặc container)
- [ ] Không có secrets hard-coded trong runner service hoặc environment
- [ ] Runner binary được xác minh SHA-256 sau khi tải

### Mạng

- [ ] Runner ở network segment riêng (VPC subnet hoặc K8s namespace với NetworkPolicy)
- [ ] Firewall chặn tất cả inbound connections
- [ ] Egress chỉ cho phép GitHub domains và services build cần thiết
- [ ] Egress proxy (nếu có) kiểm soát và log traffic

### Access Control

- [ ] Chỉ dùng cho private repositories
- [ ] Runner groups restrict repositories được phép dùng
- [ ] `restricted_to_workflows` = true cho production runners
- [ ] PR approval required trước khi chạy workflows từ fork

### Monitoring

- [ ] Alerting khi runner offline hơn 5 phút
- [ ] Alerting khi job chạy quá thời gian bình thường (potential crypto mining)
- [ ] Audit logs được forward đến SIEM (Security Information and Event Management)
- [ ] Review audit logs hàng tuần

### Bảo Trì

- [ ] Runner binary được cập nhật trong vòng 30 ngày kể từ release
- [ ] OS patching lên lịch hàng tuần
- [ ] Review runner groups và permissions hàng quý

---

**Tiếp Theo:** [4-maintenance-monitoring.md](./4-maintenance-monitoring.md) — Bảo trì và giám sát runners
