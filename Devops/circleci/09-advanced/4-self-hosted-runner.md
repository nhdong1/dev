# Self-Hosted Runner — Máy Chạy Tự Quản Lý

> Self-Hosted Runner — Máy Chạy Tự Quản Lý là agent phần mềm cài trên hạ tầng của tổ chức (on-premise — tại chỗ, cloud riêng, hoặc hybrid — kết hợp), nhận job từ CircleCI cloud và thực thi cục bộ. Giải pháp này cho phép tổ chức kiểm soát hoàn toàn môi trường thực thi trong khi vẫn tận dụng được toàn bộ tính năng quản lý pipeline của CircleCI.

---

## 📋 Mục Lục

1. [Self-Hosted Runner Là Gì?](#1-self-hosted-runner-là-gì)
2. [Khi Nào Cần Self-Hosted Runner?](#2-khi-nào-cần-self-hosted-runner)
3. [Kiến Trúc Hệ Thống](#3-kiến-trúc-hệ-thống)
4. [Cài Đặt Runner](#4-cài-đặt-runner)
5. [Cấu Hình Pipeline Dùng Runner](#5-cấu-hình-pipeline-dùng-runner)
6. [Resource Class Tùy Chỉnh](#6-resource-class-tùy-chỉnh)
7. [Vận Hành Và Bảo Trì](#7-vận-hành-và-bảo-trì)
8. [Bảo Mật Self-Hosted Runner](#8-bảo-mật-self-hosted-runner)
9. [Machine Runner vs Container Runner](#9-machine-runner-vs-container-runner)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Self-Hosted Runner Là Gì?

CircleCI có hai loại môi trường thực thi job:

```
┌─────────────────────────────────────────────────────────┐
│  CircleCI Cloud Executors (mặc định)                    │
│  - CircleCI quản lý và bảo trì toàn bộ                  │
│  - Môi trường sạch mỗi job (ephemeral — thoáng qua)     │
│  - Tính phí theo credit và resource class               │
│  - Server của CircleCI, không phải của bạn             │
└─────────────────────────────────────────────────────────┘

                        VS

┌─────────────────────────────────────────────────────────┐
│  Self-Hosted Runner (runner tự quản lý)                 │
│  - Bạn cài và quản lý agent trên máy của bạn           │
│  - CircleCI gửi job, runner thực thi cục bộ            │
│  - Không tính phí per-credit cho compute               │
│  - Bạn kiểm soát: OS, phần cứng, network, data        │
└─────────────────────────────────────────────────────────┘
```

### Luồng Hoạt Động

```
1. Developer push code → CircleCI nhận trigger
2. CircleCI phân tích config, xác định job cần runner
3. CircleCI gửi job task vào queue — hàng đợi
4. Runner agent (trên máy của bạn) polling — liên tục kiểm tra queue
5. Runner nhận job, thực thi cục bộ trên máy của bạn
6. Runner báo cáo kết quả (log, artifacts, status) về CircleCI
7. CircleCI hiển thị kết quả trên dashboard như bình thường
```

---

## 2. Khi Nào Cần Self-Hosted Runner?

### Trường Hợp Bắt Buộc Dùng

| Yêu Cầu | Lý Do Cần Runner |
| -------- | ---------------- |
| **Compliance & Regulatory** — Tuân thủ quy định | Code không được rời khỏi mạng nội bộ (PCI-DSS, HIPAA, ISO 27001) |
| **Phần Cứng Đặc Biệt** | GPU cho ML/AI training, FPGA, ARM64 native, bare metal |
| **Truy Cập Mạng Nội Bộ** | Database, API, service chỉ có trong internal network |
| **Environment Tồn Tại Lâu** | Dependencies nặng (không muốn download mỗi job) |
| **License Phần Mềm** | Phần mềm có license gắn với MAC address hoặc machine cụ thể |
| **Tối Ưu Chi Phí Ở Scale Lớn** | Khi credit cost vượt quá chi phí tự vận hành server |

### Trường Hợp Nên Dùng Cloud Executor

- Dự án nhỏ, không có yêu cầu đặc biệt về security
- Môi trường build tiêu chuẩn (Linux/macOS/Windows phổ biến)
- Không muốn overhead vận hành infrastructure
- Team nhỏ, ưu tiên tốc độ setup hơn tối ưu chi phí

---

## 3. Kiến Trúc Hệ Thống

```
┌─────────────────────────────────────────────────────────────┐
│  CircleCI Cloud                                             │
│  ┌─────────────┐    ┌───────────────┐    ┌──────────────┐  │
│  │  Pipeline   │───▶│  Job Queue    │    │  Dashboard   │  │
│  │  Scheduler  │    │  (per runner  │    │  & API       │  │
│  └─────────────┘    │  resource     │    └──────┬───────┘  │
│                     │  class)       │           │          │
│                     └───────┬───────┘           │          │
└───────────────────────────┬─┘───────────────────┘──────────┘
                            │ HTTPS polling (outbound only)
                            │ Không cần inbound port mở
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Your Infrastructure (on-premise / private cloud)          │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Runner Agent Machine 1 (linux-gpu-runner)           │  │
│  │  ┌─────────────────┐  ┌──────────────────────────┐  │  │
│  │  │  circleci-agent │  │  Job Execution Sandbox   │  │  │
│  │  │  (polling loop) │  │  - Workspace isolation   │  │  │
│  │  └────────┬────────┘  │  - Env var injection     │  │  │
│  │           │ receives  │  - Artifact collection   │  │  │
│  │           └──────────▶│                          │  │  │
│  │                       └──────────────────────────┘  │  │
│  │  Hardware: NVIDIA A100 GPU, 128GB RAM                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Runner Agent Machine 2 (linux-arm-runner)           │  │
│  │  Hardware: AWS Graviton3, ARM64 native               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Điểm quan trọng:** Runner chỉ kết nối OUTBOUND — ra ngoài đến CircleCI. Không cần mở port inbound — đường vào trên firewall của bạn.

---

## 4. Cài Đặt Runner

### Bước 1: Tạo Resource Class Trên CircleCI

```bash
# Dùng CircleCI CLI
circleci runner resource-class create \
  <your-namespace>/<runner-name> \
  "Mô tả runner" \
  --generate-token

# Ví dụ:
circleci runner resource-class create \
  mycompany/linux-gpu \
  "GPU runner for ML training jobs" \
  --generate-token

# Lưu lại token được sinh ra — chỉ hiện một lần!
# Token này dùng để xác thực runner với CircleCI
```

### Bước 2: Cài Đặt Trên Linux

```bash
# Tạo user chạy runner (không nên dùng root)
sudo useradd --uid 1500 --system --create-home circleci

# Tạo thư mục cần thiết
sudo mkdir -p /var/opt/circleci
sudo mkdir -p /opt/circleci
sudo chown -R circleci /var/opt/circleci /opt/circleci

# Download circleci-agent binary
AGENT_VERSION=$(curl https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/release.txt)
sudo curl -o /opt/circleci/circleci-launch-agent \
  "https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/${AGENT_VERSION}/linux/amd64/circleci-launch-agent"
sudo chmod +x /opt/circleci/circleci-launch-agent

# Kiểm tra cài đặt
/opt/circleci/circleci-launch-agent --version
```

### Bước 3: Cấu Hình Runner

```yaml
# /etc/opt/circleci/launch-agent-config.yaml
api:
  auth_token: <TOKEN_TỪ_BƯỚC_1>

runner:
  name: "gpu-runner-01"                    # Tên định danh runner instance
  command_prefix: ["sudo", "-niHu", "circleci"]  # Chạy dưới user circleci
  working_directory: /var/opt/circleci/workdir/%s
  cleanup_working_directory: true           # Dọn dẹp sau mỗi job

logging:
  file: /var/log/circleci/circleci-launch-agent.log
  level: info
```

### Bước 4: Cài Đặt Systemd Service — Dịch Vụ Hệ Thống

```ini
# /usr/lib/systemd/system/circleci.service
[Unit]
Description=CircleCI Runner — Máy Chạy CircleCI
After=network.target

[Service]
Type=simple
User=circleci
ExecStart=/opt/circleci/circleci-launch-agent \
    --config /etc/opt/circleci/launch-agent-config.yaml
Restart=always
RestartSec=5
KillMode=process
TimeoutStopSec=5m

[Install]
WantedBy=multi-user.target
```

```bash
# Kích hoạt và khởi động service
sudo systemctl enable circleci
sudo systemctl start circleci

# Kiểm tra trạng thái
sudo systemctl status circleci
sudo journalctl -u circleci -f  # Xem log real-time
```

### Cài Đặt Trên macOS

```bash
# Download agent
AGENT_VERSION=$(curl https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/release.txt)
curl -o /opt/circleci/circleci-launch-agent \
  "https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/${AGENT_VERSION}/darwin/amd64/circleci-launch-agent"
chmod +x /opt/circleci/circleci-launch-agent

# Tạo launchd plist — file dịch vụ macOS
cat > /Library/LaunchDaemons/com.circleci.runner.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.circleci.runner</string>
    <key>ProgramArguments</key>
    <array>
      <string>/opt/circleci/circleci-launch-agent</string>
      <string>--config</string>
      <string>/Library/Preferences/com.circleci.runner.yaml</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
  </dict>
</plist>
EOF

sudo launchctl load /Library/LaunchDaemons/com.circleci.runner.plist
```

### Cài Đặt Bằng Docker (Container Runner)

```bash
# Dùng Docker để chạy runner (Container Runner mode)
docker run -d \
  --name circleci-runner \
  --restart always \
  -e CIRCLECI_RUNNER_NAME="docker-runner-01" \
  -e CIRCLECI_RUNNER_AUTH_TOKEN="<TOKEN>" \
  -e CIRCLECI_RUNNER_API_URL="https://runner.circleci.com" \
  -v /var/run/docker.sock:/var/run/docker.sock \  # Để chạy Docker-in-Docker
  circleci/runner:latest
```

---

## 5. Cấu Hình Pipeline Dùng Runner

### Syntax Trong config.yml

```yaml
version: 2.1

jobs:
  # Job chạy trên cloud executor (mặc định)
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm run build

  # Job chạy trên self-hosted runner
  ml-training:
    machine: true  # Bắt buộc khai báo machine: true với runner
    resource_class: mycompany/linux-gpu  # Namespace/resource-class-name
    steps:
      - checkout
      - run:
          name: Chạy ML training với GPU
          command: |
            nvidia-smi  # Kiểm tra GPU
            python train.py --epochs 100 --device cuda
      - store_artifacts:
          path: models/

  # Job khác trên runner khác
  integration-test:
    machine: true
    resource_class: mycompany/linux-arm
    steps:
      - checkout
      - run: ./run-integration-tests.sh

workflows:
  ci-pipeline:
    jobs:
      - build         # Cloud executor
      - ml-training:  # Self-hosted runner
          requires:
            - build
      - integration-test:  # Self-hosted runner khác
          requires:
            - build
```

### Runner Với Docker Executor Style

```yaml
jobs:
  secure-deploy:
    # Dùng runner nhưng vẫn có thể chạy container bên trong
    machine: true
    resource_class: mycompany/secure-runner
    steps:
      - checkout
      - run:
          name: Deploy với Docker trong runner
          command: |
            # Runner đã có Docker cài sẵn
            docker build -t myapp:latest .
            docker push internal-registry.company.com/myapp:latest
            kubectl apply -f k8s/ --kubeconfig /etc/k8s/internal-config
```

---

## 6. Resource Class Tùy Chỉnh

### Namespace — Không Gian Tên Và Resource Class

```
Format: <namespace>/<resource-class-name>

Namespace = thường là tên tổ chức trên CircleCI
Resource Class = tên bạn đặt để phân loại runner

Ví dụ:
  acmecorp/linux-gpu-large    → Runner Linux có GPU lớn
  acmecorp/linux-arm64        → Runner ARM64 native
  acmecorp/windows-dotnet     → Runner Windows có .NET
  acmecorp/macos-xcode15      → Runner macOS với Xcode 15
  acmecorp/secure-deploy      → Runner trong mạng secure
```

### Quản Lý Resource Class Qua CLI

```bash
# Liệt kê tất cả resource class
circleci runner resource-class list

# Xem chi tiết một resource class
circleci runner resource-class get mycompany/linux-gpu

# Tạo token mới cho resource class (khi token cũ bị lộ)
circleci runner token create mycompany/linux-gpu "Renewed 2026-05"

# Xóa token cũ
circleci runner token delete <token-id>

# Xem trạng thái các runner đang active
circleci runner instance list mycompany/linux-gpu
```

---

## 7. Vận Hành Và Bảo Trì

### Scaling Runner — Mở Rộng Số Lượng Runner

```bash
# Chạy nhiều runner instance trên cùng một máy (nếu máy đủ mạnh)
# Mỗi instance cần tên riêng

# Runner instance 1
cat > /etc/opt/circleci/config-1.yaml << EOF
api:
  auth_token: <SAME_TOKEN>
runner:
  name: gpu-runner-01-instance-1
  working_directory: /var/opt/circleci/workdir/1/%s
EOF

# Runner instance 2
cat > /etc/opt/circleci/config-2.yaml << EOF
api:
  auth_token: <SAME_TOKEN>
runner:
  name: gpu-runner-01-instance-2
  working_directory: /var/opt/circleci/workdir/2/%s
EOF
```

### Monitoring Runner Health — Giám Sát Sức Khỏe Runner

```bash
# Kiểm tra runner đang online không
circleci runner instance list <namespace>/<resource-class>

# Xem log lỗi
sudo journalctl -u circleci --since "1 hour ago" | grep -i error

# Kiểm tra job đang chạy
circleci runner job get <job-id>

# Health check script đơn giản
#!/bin/bash
# /opt/circleci/health-check.sh
STATUS=$(systemctl is-active circleci)
if [ "$STATUS" != "active" ]; then
    echo "ALERT: CircleCI runner không chạy!" | \
      mail -s "Runner Down" ops-team@company.com
    systemctl restart circleci
fi
```

### Cập Nhật Runner Agent

```bash
# Kiểm tra version hiện tại
/opt/circleci/circleci-launch-agent --version

# Download version mới
NEW_VERSION=$(curl https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/release.txt)
sudo curl -o /opt/circleci/circleci-launch-agent-new \
  "https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/${NEW_VERSION}/linux/amd64/circleci-launch-agent"

# Graceful update — cập nhật không gián đoạn
sudo systemctl stop circleci
sudo mv /opt/circleci/circleci-launch-agent-new /opt/circleci/circleci-launch-agent
sudo chmod +x /opt/circleci/circleci-launch-agent
sudo systemctl start circleci
```

### Cleanup Chiến Lược — Dọn Dẹp

```yaml
# Trong launch-agent-config.yaml
runner:
  cleanup_working_directory: true   # Xóa workdir sau mỗi job
  max_run_time: 3600               # Timeout job tối đa (giây)

# Cron job dọn dẹp artifact cũ
# Thêm vào crontab của user circleci:
# 0 2 * * * find /var/opt/circleci -type f -mtime +7 -delete
```

---

## 8. Bảo Mật Self-Hosted Runner

### Network Security — Bảo Mật Mạng

```
Runner chỉ cần kết nối outbound đến:
  *.circleci.com:443 (HTTPS)
  *.circleci-binary-releases.s3.amazonaws.com:443

KHÔNG cần:
  - Mở port inbound — đường vào
  - DMZ hoặc public IP
  - Proxy ngược — reverse proxy phức tạp
```

```bash
# Firewall rules tối thiểu (ví dụ với iptables)
# Chỉ cho phép outbound HTTPS
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT
iptables -A OUTPUT -j DROP  # Block tất cả traffic khác outbound
```

### Isolation — Cô Lập Môi Trường

```yaml
# Chiến lược cô lập job bằng Docker
runner:
  # Mỗi job chạy trong container riêng biệt
  working_directory: /var/opt/circleci/workdir/%s
  cleanup_working_directory: true

# Trong job, dùng Docker để thêm lớp cô lập
steps:
  - run:
      name: Chạy trong container cô lập
      command: |
        docker run --rm \
          --network none \  # Không có network (nếu không cần)
          -v $(pwd):/workspace \
          myapp-builder:latest \
          ./run-tests.sh
```

### Secret Management — Quản Lý Bí Mật

```bash
# Secrets được inject qua CircleCI Contexts — vẫn an toàn
# Runner nhận secrets từ CircleCI, không lưu trên máy thường xuyên

# KHÔNG làm:
# - Hardcode secrets trong config file của runner
# - Lưu secrets trong environment variable thường trực trên runner

# NÊN làm:
# - Dùng CircleCI Contexts để inject secrets vào job
# - Dùng Vault/AWS Secrets Manager trên internal network
# - Rotate runner auth token định kỳ (3-6 tháng)
```

### Audit — Kiểm Toán

```bash
# Log mọi job chạy trên runner
sudo journalctl -u circleci --output=json | \
  jq 'select(.MESSAGE | contains("job"))' >> /var/log/circleci/job-audit.log

# Monitor unusual patterns — mẫu bất thường
# Ví dụ: alert khi job chạy quá 2 giờ
```

---

## 9. Machine Runner vs Container Runner

CircleCI hỗ trợ hai loại runner agent:

### Machine Runner — Runner Máy Vật Lý/VM

```
- Agent chạy trực tiếp trên OS của máy
- Job thực thi trong môi trường của máy đó
- Có thể chạy Docker, GPU, special hardware
- Workdir được dọn dẹp sau mỗi job
- Phù hợp: GPU, bare metal, macOS, Windows
```

### Container Runner — Runner Dạng Container

```
- Agent chạy trên Kubernetes cluster
- Mỗi job chạy trong một Pod riêng biệt
- Hoàn toàn ephemeral — sạch sẽ mỗi job
- Dễ scale horizontal — mở rộng theo chiều ngang
- Phù hợp: Kubernetes native, scale linh hoạt
```

```yaml
# Container Runner configuration — Cấu hình trên Kubernetes
# circleci-container-runner.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: circleci-runner
  namespace: circleci
spec:
  replicas: 3  # 3 runner instances chạy song song
  selector:
    matchLabels:
      app: circleci-runner
  template:
    spec:
      containers:
        - name: runner
          image: circleci/runner:latest
          env:
            - name: CIRCLECI_RUNNER_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: circleci-runner-token
                  key: token
            - name: CIRCLECI_RUNNER_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # Dùng pod name làm runner name
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Self-Hosted Runner là gì và khi nào nên dùng?**

> Self-Hosted Runner là agent phần mềm cài trên hạ tầng của tổ chức, nhận job từ CircleCI và thực thi cục bộ. Nên dùng khi: (1) compliance yêu cầu code không rời mạng nội bộ, (2) cần phần cứng đặc biệt như GPU hoặc ARM, (3) job cần truy cập resource chỉ có trong internal network, (4) chi phí cloud compute quá cao ở scale lớn.

**Q: Runner kết nối với CircleCI cloud như thế nào? Có cần mở inbound port không?**

> Không cần mở inbound port. Runner sử dụng mô hình polling — liên tục kiểm tra — outbound HTTPS đến CircleCI API. Runner chủ động gọi ra ngoài để lấy job, không phải CircleCI gọi vào runner. Chỉ cần firewall cho phép outbound HTTPS (port 443) đến `*.circleci.com`.

**Q: Sự khác biệt giữa Machine Runner và Container Runner?**

> Machine Runner chạy trực tiếp trên OS của máy vật lý hoặc VM, phù hợp với GPU, bare metal, macOS. Container Runner chạy trên Kubernetes, mỗi job là một Pod riêng, hoàn toàn ephemeral và dễ scale. Container Runner phù hợp với môi trường cloud-native, Machine Runner phù hợp với yêu cầu phần cứng đặc biệt.

**Q: Làm thế nào để scale self-hosted runner khi có nhiều job chờ?**

> Có hai cách: (1) Chạy nhiều runner instance trên cùng một máy (nếu đủ tài nguyên) bằng cách có nhiều service với config khác nhau. (2) Dùng Container Runner trên Kubernetes với HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang để scale theo số job trong queue. Có thể kết hợp CircleCI runner metrics với Kubernetes metrics để trigger auto-scaling.

**Q: Bảo mật runner như thế nào để tránh job độc hại chạy trên infrastructure?**

> (1) Dùng Restricted Contexts — Ngữ Cảnh Giới Hạn để chỉ cho phép trusted project dùng runner. (2) Cô lập mỗi job trong Docker container riêng. (3) Dùng user không có quyền root, hạn chế filesystem permissions. (4) Rotate runner auth token định kỳ. (5) Network firewall chặt chẽ, chỉ cho phép traffic cần thiết. (6) Audit log — nhật ký kiểm toán mọi job chạy trên runner.

---

## 🔗 Xem Thêm

- [5-monorepo-strategy.md](5-monorepo-strategy.md) — Dùng runner trong chiến lược monorepo
- [06-security/1-contexts.md](../06-security/1-contexts.md) — Bảo mật secrets với Contexts
- [CircleCI Docs: Self-Hosted Runners](https://circleci.com/docs/runner-overview/)

---

**Cập Nhật Lần Cuối:** 2026-05-20
