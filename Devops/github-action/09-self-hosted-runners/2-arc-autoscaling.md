# ARC — Actions Runner Controller & Auto-scaling trên Kubernetes

> ARC — Actions Runner Controller (Bộ Điều Khiển Runner Hành Động) — một Kubernetes Operator (Toán Tử Kubernetes) cho phép tự động mở rộng và thu hẹp số lượng self-hosted runners dựa trên số lượng jobs đang chờ, bao gồm khả năng scale-to-zero (thu về không runner nào khi không có việc).

---

## 📚 Mục Lục

1. [ARC Là Gì](#arc-là-gì)
2. [Kiến Trúc ARC](#kiến-trúc-arc)
3. [Cài Đặt ARC Bằng Helm](#cài-đặt-arc-bằng-helm)
4. [RunnerScaleSet — Bộ Mở Rộng Runner](#runnerscaleset)
5. [Ephemeral Runners Trên Kubernetes](#ephemeral-runners-trên-kubernetes)
6. [Cấu Hình Nâng Cao](#cấu-hình-nâng-cao)
7. [Monitoring ARC](#monitoring-arc)
8. [Troubleshooting](#troubleshooting)
9. [Bài Tập Thực Hành](#bài-tập-thực-hành)

---

## 🤔 ARC Là Gì

### Vấn Đề ARC Giải Quyết

Với self-hosted runners truyền thống (VM cố định):
- Lãng phí tài nguyên khi không có job
- Không đủ runners khi có spike (đột biến) traffic
- Phải quản lý thủ công số lượng VMs
- State bị tích lũy giữa các jobs (rủi ro bảo mật)

ARC giải quyết tất cả bằng cách chạy mỗi runner như một **Kubernetes Pod** (Đơn Vị Tính Toán Kubernetes) tạm thời.

### Luồng Hoạt Động

```
1. Workflow trigger → Job đưa vào queue chờ
2. ARC Controller phát hiện job đang chờ
3. ARC tạo Pod mới (runner) trên Kubernetes
4. Runner Pod nhận job, chạy đến khi hoàn thành
5. Runner Pod tự xóa sau khi job kết thúc
6. Khi không có job → ARC scale về 0 Pods
```

### Hai Loại Runner Pod

| Loại | Mô Tả | Dùng Khi |
|---|---|---|
| **Kubernetes mode** | Runner và containers của step chạy trong một Pod | Đơn giản, ít tùy chỉnh |
| **Docker-in-Docker (DinD)** | Runner chạy Docker daemon bên trong Pod | Cần `docker build`, `docker compose` |
| **Docker outside of Docker (DooD)** | Mount Docker socket từ host vào Pod | Ít cô lập hơn, nhanh hơn |

---

## 🏗️ Kiến Trúc ARC

```
┌──────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             arc-systems namespace                         │    │
│  │                                                           │    │
│  │  ┌──────────────────────────────────────────────────┐    │    │
│  │  │  AutoscalingRunnerSet Controller                  │    │    │
│  │  │  (Bộ Điều Khiển Mở Rộng Runner Tự Động)         │    │    │
│  │  │                                                    │    │    │
│  │  │  - Watch GitHub API cho jobs đang chờ             │    │    │
│  │  │  - Tạo/xóa EphemeralRunner (Pod)                  │    │    │
│  │  │  - Quản lý lifecycle của runner fleet             │    │    │
│  │  └──────────────────────────────────────────────────┘    │    │
│  │                                                           │    │
│  │  ┌────────────────────────────────────────────────────┐  │    │
│  │  │  Listener Pod (Pod Lắng Nghe)                      │  │    │
│  │  │  - Long poll GitHub Actions Service                 │  │    │
│  │  │  - Thông báo cho Controller khi có job mới         │  │    │
│  │  └────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             arc-runners namespace                         │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│  │  │ Runner   │  │ Runner   │  │ Runner   │  ← Pods tạm   │    │
│  │  │ Pod 1    │  │ Pod 2    │  │ Pod 3    │    thời       │    │
│  │  │ (Job A)  │  │ (Job B)  │  │ (Job C)  │               │    │
│  │  └──────────┘  └──────────┘  └──────────┘               │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                                 │ HTTPS outbound
                    ┌────────────▼─────────────┐
                    │        GitHub.com         │
                    │  Actions Service API      │
                    └──────────────────────────┘
```

### Components Chính

| Component | Vai Trò |
|---|---|
| **AutoscalingRunnerSet Controller** | Kubernetes Operator điều phối toàn bộ lifecycle runner |
| **Listener Pod** | Nhận job assignments từ GitHub, báo cho Controller |
| **EphemeralRunner Pod** | Pod chạy actual job, tự xóa sau khi xong |
| **RunnerScaleSet** (CRD) | Custom Resource — khai báo cấu hình runners |

---

## 🚀 Cài Đặt ARC Bằng Helm

### Yêu Cầu

- Kubernetes 1.25+ (khuyến nghị 1.28+)
- Helm 3.8+
- `kubectl` đã cấu hình trỏ vào cluster đích
- GitHub Personal Access Token (PAT) hoặc GitHub App

### Bước 1: Thêm Helm Repository

```bash
# Thêm ARC Helm chart repository
helm repo add actions-runner-controller \
  https://actions-runner-controller.github.io/actions-runner-controller

# Cập nhật cache
helm repo update

# Kiểm tra chart có sẵn
helm search repo actions-runner-controller
```

### Bước 2: Cài Đặt ARC Controller

```bash
# Tạo namespace riêng cho ARC system
kubectl create namespace arc-systems

# Cài đặt controller
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version 0.9.3    # Thay bằng phiên bản mới nhất
```

### Bước 3: Tạo Secret Xác Thực

#### Tùy Chọn A: Personal Access Token (PAT)

```bash
# PAT cần scope: repo (nếu repo runner) hoặc admin:org (nếu org runner)
kubectl create secret generic arc-github-secret \
  --namespace arc-systems \
  --from-literal=github_token="ghp_xxxxxxxxxxxxxxxxxxxx"
```

#### Tùy Chọn B: GitHub App (Khuyến Nghị Cho Production)

```bash
# Tạo Kubernetes Secret từ GitHub App credentials
kubectl create secret generic arc-github-app-secret \
  --namespace arc-systems \
  --from-literal=github_app_id="<APP_ID>" \
  --from-literal=github_app_installation_id="<INSTALLATION_ID>" \
  --from-literal=github_app_private_key="$(cat private-key.pem)"
```

> **GitHub App vs PAT:** GitHub App cấp quyền chi tiết hơn (fine-grained), tự động rotate credentials, và audit log rõ ràng hơn. Luôn dùng GitHub App cho production.

---

## 📐 RunnerScaleSet

RunnerScaleSet — Custom Resource Definition (Định Nghĩa Tài Nguyên Tùy Chỉnh) khai báo một nhóm runners với cấu hình cụ thể.

### ScaleSet Cơ Bản

```yaml
# arc-runner-set.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: arc-runner-set
  namespace: arc-runners
---
# Hoặc cài trực tiếp qua helm install:
```

```bash
# Tạo namespace cho runners
kubectl create namespace arc-runners

# Cài đặt RunnerScaleSet
helm install arc-runner-set \
  --namespace arc-runners \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/{owner}/{repo}" \
  --set githubConfigSecret="arc-github-secret" \
  --set controllerServiceAccount.namespace="arc-systems" \
  --set controllerServiceAccount.name="arc-gha-rs-controller" \
  --set runnerScaleSetName="arc-runner-set" \
  --set minRunners=0 \
  --set maxRunners=10
```

### Values File Đầy Đủ

```yaml
# arc-runner-values.yaml
githubConfigUrl: "https://github.com/{owner}/{repo}"
# Hoặc cho organization: "https://github.com/{org}"
# Hoặc cho enterprise:   "https://github.com/enterprises/{enterprise}"

githubConfigSecret: "arc-github-secret"

controllerServiceAccount:
  namespace: arc-systems
  name: arc-gha-rs-controller

# Tên ScaleSet — dùng trong workflow runs-on
runnerScaleSetName: "arc-k8s-runner"

# Scale policy (Chính Sách Mở Rộng)
minRunners: 0      # Scale to zero khi không có job
maxRunners: 20     # Giới hạn tối đa runners cùng lúc

# Template cho mỗi Runner Pod
template:
  spec:
    # SecurityContext — Ngữ Cảnh Bảo Mật
    securityContext:
      runAsNonRoot: true
      runAsUser: 1001
      runAsGroup: 1001
      fsGroup: 1001

    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]

        # Resource requests và limits
        resources:
          requests:
            cpu: "500m"         # 0.5 CPU core
            memory: "512Mi"
          limits:
            cpu: "2000m"        # 2 CPU cores
            memory: "4Gi"

        env:
          - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
            value: "false"

    # Tự động dọn dẹp Pod sau khi hoàn thành
    restartPolicy: Never

    # Chạy trên nodes có label cụ thể (Node Selector — Bộ Chọn Node)
    nodeSelector:
      node-role: runner

    # Tolerations — cho phép chạy trên nodes có taints
    tolerations:
      - key: "runner-only"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```

### Sử Dụng Trong Workflow

```yaml
# .github/workflows/arc-job.yml
name: ARC Runner Job
on: push

jobs:
  build:
    # runs-on trỏ đến tên RunnerScaleSet
    runs-on: arc-k8s-runner
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm ci && npm run build
```

---

## 🐳 Ephemeral Runners Trên Kubernetes

### Docker-in-Docker (DinD) Mode

Dành cho jobs cần chạy Docker commands (`docker build`, `docker push`).

```yaml
# arc-dind-values.yaml
template:
  spec:
    initContainers:
      - name: init-dind-externals
        image: ghcr.io/actions/actions-runner:latest
        command:
          - cp
          - -r
          - /home/runner/externals/.
          - /home/runner/tmpDir/
        volumeMounts:
          - name: dind-externals
            mountPath: /home/runner/tmpDir

    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        env:
          - name: DOCKER_HOST
            value: "unix:///var/run/docker.sock"
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
          - name: dind-externals
            mountPath: /home/runner/externals

      # DinD sidecar container
      - name: dind
        image: docker:dind
        securityContext:
          privileged: true    # DinD yêu cầu privileged mode
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
          - name: dind-externals
            mountPath: /home/runner/externals

    volumes:
      - name: work
        emptyDir: {}
      - name: dind-sock
        emptyDir: {}
      - name: dind-externals
        emptyDir: {}
```

> **Lưu ý bảo mật:** DinD yêu cầu `privileged: true` — container có quyền tương đương root trên host. Chỉ dùng trên nodes chuyên dụng cho CI, không chia sẻ với workloads production.

### Container Mode (Kubernetes Native)

Mỗi step container (`container:` trong workflow) chạy như một Pod riêng biệt — không cần DinD.

```yaml
# Workflow sử dụng container mode
jobs:
  build:
    runs-on: arc-k8s-runner
    container:
      image: node:20-alpine
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

---

## ⚙️ Cấu Hình Nâng Cao

### Persistent Volume Cho Cache

```yaml
# Dùng PersistentVolumeClaim để cache npm packages, Maven repo, v.v.
template:
  spec:
    containers:
      - name: runner
        volumeMounts:
          - name: npm-cache
            mountPath: /home/runner/.npm

    volumes:
      - name: npm-cache
        persistentVolumeClaim:
          claimName: runner-npm-cache
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: runner-npm-cache
  namespace: arc-runners
spec:
  accessModes: [ReadWriteMany]    # Chia sẻ giữa nhiều runner Pods
  storageClassName: efs-sc        # AWS EFS hoặc tương đương
  resources:
    requests:
      storage: 50Gi
```

### Multi ScaleSet Cho Nhiều Workloads

```bash
# ScaleSet cho general builds
helm install arc-runner-general \
  --namespace arc-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set runnerScaleSetName="k8s-runner-general" \
  --set minRunners=0 \
  --set maxRunners=20 \
  --values general-runner-values.yaml

# ScaleSet cho GPU jobs
helm install arc-runner-gpu \
  --namespace arc-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set runnerScaleSetName="k8s-runner-gpu" \
  --set minRunners=0 \
  --set maxRunners=5 \
  --values gpu-runner-values.yaml
```

```yaml
# gpu-runner-values.yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        resources:
          limits:
            nvidia.com/gpu: 1    # Yêu cầu 1 GPU
    nodeSelector:
      accelerator: nvidia-tesla-v100
```

### Webhook-based Scaling (Mở Rộng Dựa Trên Webhook)

Thay vì polling, ARC có thể nhận webhooks từ GitHub để scale nhanh hơn.

```yaml
# Kích hoạt webhook scaling trong values
webhookDrivenScaling:
  enabled: true
  webhookSecretName: "arc-webhook-secret"

# Cần expose một endpoint để GitHub gửi webhook
# Tạo Ingress trỏ vào ARC webhook service
```

---

## 📊 Monitoring ARC

### Kiểm Tra Trạng Thái Cơ Bản

```bash
# Xem tất cả runners đang chạy
kubectl get pods -n arc-runners

# Xem ScaleSet và số lượng runners
kubectl get autoscalingrunnerset -n arc-runners

# Xem Listener Pod
kubectl get pods -n arc-systems

# Logs của Controller
kubectl logs -n arc-systems \
  -l app.kubernetes.io/component=controller-manager \
  --follow

# Logs của một Runner Pod cụ thể
kubectl logs -n arc-runners <runner-pod-name>
```

### Prometheus Metrics

ARC xuất metrics qua Prometheus endpoint.

```bash
# Port forward để xem metrics
kubectl port-forward -n arc-systems svc/arc-gha-rs-controller-metrics-service 8080:8080

# Xem metrics
curl http://localhost:8080/metrics | grep actions_
```

**Metrics quan trọng:**

| Metric | Mô Tả |
|---|---|
| `actions_github_app_rate_limit_remaining` | GitHub API rate limit còn lại |
| `actions_runner_pods_total` | Tổng số runner Pods |
| `actions_runner_pods_starting` | Pods đang khởi động |
| `actions_runner_pods_running` | Pods đang chạy job |

### Grafana Dashboard

```yaml
# grafana-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: arc-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  arc-dashboard.json: |
    {
      "title": "ARC Runner Dashboard",
      "panels": [
        {
          "title": "Active Runners",
          "targets": [{"expr": "actions_runner_pods_running"}]
        },
        {
          "title": "Pending Jobs",
          "targets": [{"expr": "actions_runner_pods_starting"}]
        }
      ]
    }
```

---

## 🔧 Troubleshooting

### Runner Pod Không Khởi Động

```bash
# Kiểm tra events trên Pod
kubectl describe pod <pod-name> -n arc-runners

# Nguyên nhân thường gặp:
# 1. ImagePullBackOff — không pull được image
# 2. Pending — không có node đủ tài nguyên
# 3. CrashLoopBackOff — runner crash sau khi start
```

### Runner Online Nhưng Job Không Được Assign

```bash
# Kiểm tra logs của Listener Pod
kubectl logs -n arc-systems \
  -l app.kubernetes.io/component=runner-scale-set-listener

# Kiểm tra GitHub App permissions
# Cần: Actions: Read & Write, Administration: Read & Write
```

### Scale Không Hoạt Động

```bash
# Kiểm tra Controller logs
kubectl logs -n arc-systems \
  deployment/arc-gha-rs-controller

# Kiểm tra AutoscalingRunnerSet status
kubectl describe autoscalingrunnerset -n arc-runners

# Thường do:
# - GitHub API rate limiting
# - Token/GitHub App hết hạn
# - minRunners > maxRunners
```

### Lỗi Phổ Biến Và Cách Xử Lý

| Lỗi | Nguyên Nhân | Cách Xử Lý |
|---|---|---|
| `Failed to acquire job` | Token hết hạn | Rotate PAT hoặc GitHub App credentials |
| `Runner is already claimed` | Runner bị duplicate | Xóa runner cũ trong GitHub Settings |
| `No runners matching labels` | `runs-on` không match ScaleSet name | Kiểm tra `runnerScaleSetName` trong values |
| Pod stuck `Pending` | Không đủ resources | Tăng node pool hoặc giảm resource requests |

---

## 🧪 Bài Tập Thực Hành

### Bài 1: Cài Đặt ARC Trên Minikube

```bash
# Khởi động Minikube
minikube start --cpus 4 --memory 8192

# Cài ARC Controller
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# Tạo secret (dùng PAT thực)
kubectl create secret generic arc-github-secret \
  --namespace arc-systems \
  --from-literal=github_token="ghp_your_actual_token"

# Cài RunnerScaleSet
helm install arc-runner-set \
  --namespace arc-runners \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/{your-org}/{your-repo}" \
  --set githubConfigSecret="arc-github-secret" \
  --set runnerScaleSetName="k8s-test-runner" \
  --set minRunners=0 \
  --set maxRunners=5

# Trigger workflow và quan sát pods scaling up
kubectl get pods -n arc-runners --watch
```

### Bài 2: Workflow Dùng ARC Runner

```yaml
# .github/workflows/arc-test.yml
name: ARC Runner Test
on: workflow_dispatch

jobs:
  test:
    runs-on: k8s-test-runner    # Tên ScaleSet
    steps:
      - name: Show pod info
        run: |
          echo "Pod name: $HOSTNAME"
          echo "Node: $(cat /etc/hostname)"
          kubectl get nodes 2>/dev/null || echo "kubectl not available (expected)"
      - name: Sleep to observe scaling
        run: sleep 30
```

---

## 📋 Tóm Tắt So Sánh

| Tiêu Chí | Static VM Runners | ARC (Kubernetes) |
|---|---|---|
| **Chi phí khi nhàn rỗi** | Vẫn trả tiền VM | Scale to zero |
| **Đáp ứng spike** | Chậm (phải tạo VM mới) | Nhanh (Pod mới trong giây) |
| **Quản lý** | Phức tạp (patching VM) | Đơn giản hơn (Helm upgrade) |
| **Ephemeral by default** | Không (cần cấu hình) | Có (mỗi Pod = 1 job) |
| **Yêu cầu kỹ năng** | Linux sysadmin | Kubernetes |
| **Phù hợp với** | Legacy workloads | Cloud-native teams |

---

**Tiếp Theo:** [3-security-isolation.md](./3-security-isolation.md) — Bảo mật và cô lập runners
