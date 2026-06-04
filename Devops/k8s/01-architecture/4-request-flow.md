# Request Flow — Luồng Xử Lý Từ kubectl apply Đến Pod Chạy

> Hiểu luồng xử lý end-to-end là chìa khoá để debug hiệu quả và trả lời tốt câu hỏi phỏng vấn về kiến trúc Kubernetes.

## Mục Lục

1. [Tổng Quan Luồng](#tổng-quan-luồng)
2. [Giai Đoạn 1 — Client và API Server](#giai-đoạn-1--client-và-api-server)
3. [Giai Đoạn 2 — Controller Manager](#giai-đoạn-2--controller-manager)
4. [Giai Đoạn 3 — Scheduler](#giai-đoạn-3--scheduler)
5. [Giai Đoạn 4 — kubelet và Container Runtime](#giai-đoạn-4--kubelet-và-container-runtime)
6. [Giai Đoạn 5 — Networking và Service](#giai-đoạn-5--networking-và-service)
7. [Sơ Đồ Đầy Đủ](#sơ-đồ-đầy-đủ)
8. [Luồng Xoá Deployment](#luồng-xoá-deployment)
9. [Luồng Rolling Update](#luồng-rolling-update)
10. [Debug Mỗi Giai Đoạn](#debug-mỗi-giai-đoạn)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Luồng

Khi bạn chạy `kubectl apply -f deployment.yaml`, Kubernetes thực hiện chuỗi sự kiện phức tạp:

```
Người dùng
   │ kubectl apply -f deployment.yaml
   │
   ▼ [1] Giai đoạn Client
kubectl đọc YAML → xác định server-side apply hay client-side apply
   │
   ▼ [2] Giai đoạn API Server
Xác thực (Authentication) → Phân quyền (Authorization) → Admission Control → Lưu etcd
   │
   ▼ [3] Giai đoạn Controller Manager
Deployment Controller → ReplicaSet Controller → Tạo Pod object
   │
   ▼ [4] Giai đoạn Scheduler
Tính toán node phù hợp → Gán node cho Pod
   │
   ▼ [5] Giai đoạn kubelet
Kéo image → Tạo container → Health check → Báo cáo Ready
   │
   ▼ [6] Giai đoạn Networking
Endpoints Controller cập nhật → kube-proxy cập nhật iptables
   │
   ▼ Pod sẵn sàng nhận traffic!
```

**Thời gian tổng thể điển hình:** 15–60 giây (phần lớn thời gian là kéo image Docker)

---

## Giai Đoạn 1 — Client và API Server

### Bước 1.1: kubectl Xử Lý File YAML

```bash
kubectl apply -f deployment.yaml
```

kubectl thực hiện:
1. Đọc và parse file YAML
2. Xác định `apiVersion` và `kind` để biết gọi endpoint nào
3. Nếu object đã tồn tại: so sánh với trạng thái hiện tại (client-side diff hoặc server-side apply)
4. Gửi HTTP request đến API Server

```bash
# Xem kubectl đang gọi gì mà không thực sự apply
kubectl apply -f deployment.yaml --dry-run=server -v=8
# -v=8: verbose mode — hiển thị HTTP request/response đầy đủ
```

**API Endpoint được gọi:**
```
POST /apis/apps/v1/namespaces/default/deployments        ← Tạo mới
PUT  /apis/apps/v1/namespaces/default/deployments/nginx  ← Cập nhật hoàn toàn
PATCH /apis/apps/v1/namespaces/default/deployments/nginx ← Cập nhật một phần (apply dùng cái này)
```

### Bước 1.2: Authentication (Xác Thực)

API Server xác minh danh tính người gửi request:

```
Request đến API Server
   │
   ├─ Thử X.509 Certificate (chứng chỉ): đọc từ ~/.kube/config
   ├─ Thử Bearer Token: service account token, OIDC token
   ├─ Thử Webhook Token: gọi external auth service
   └─ Anonymous: nếu bật, request không có thông tin xác thực
```

Nếu không xác thực được: `401 Unauthorized`

```bash
# Xem thông tin xác thực hiện tại
kubectl config view --minify
kubectl auth whoami   # K8s >= 1.28
```

### Bước 1.3: Authorization (Uỷ Quyền — RBAC)

API Server kiểm tra quyền thực hiện action:

```
Subject (user/serviceaccount) muốn thực hiện:
  verb: create
  resource: deployments
  apiGroup: apps
  namespace: default

RBAC Authorizer kiểm tra:
  1. Có RoleBinding / ClusterRoleBinding nào bind Subject với Role có quyền này không?
  2. Nếu có → ALLOW
  3. Không → 403 Forbidden
```

```bash
# Kiểm tra quyền của bản thân
kubectl auth can-i create deployments --namespace=production

# Kiểm tra quyền của service account khác
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa
```

### Bước 1.4: Admission Control (Kiểm Soát Đầu Vào)

Sau khi xác thực và phân quyền, request qua chuỗi Admission Controller:

**Mutating Admission (Biến Đổi) — có thể thay đổi object:**

```
MutatingAdmissionWebhook (ví dụ: Istio inject envoy sidecar)
   → Thêm container envoy vào PodSpec
   → Thêm annotation đánh dấu đã inject

DefaultStorageClass
   → Nếu PVC không có storageClassName → tự đặt storageClass mặc định

ServiceAccount
   → Nếu Pod không có serviceAccountName → gắn ServiceAccount "default"
```

**Validating Admission (Kiểm Tra) — chỉ accept hoặc reject:**

```
NamespaceLifecycle
   → Namespace "production" có đang terminating không? → Reject nếu có

ResourceQuota
   → Namespace đã dùng hết CPU/memory quota chưa? → Reject nếu hết

PodSecurity (thay thế PodSecurityPolicy từ K8s 1.25)
   → Pod có privileged container không? Namespace chỉ cho phép Baseline?
   → Reject nếu vi phạm

ValidatingAdmissionWebhook (ví dụ: OPA Gatekeeper)
   → Validate theo policy tuỳ chỉnh: image phải có digest, không dùng latest tag
```

### Bước 1.5: Lưu Vào etcd

Sau khi qua tất cả Admission Controller:

```
API Server serialize object → protobuf (hoặc JSON)
   → Ghi vào etcd tại key: /registry/apps/deployments/default/nginx-deployment
   → etcd confirm ghi thành công (Raft quorum)
   → API Server tạo event: "Deployment nginx-deployment created"
   → API Server trả 201 Created cho kubectl
```

---

## Giai Đoạn 2 — Controller Manager

### Bước 2.1: Deployment Controller Phát Hiện Deployment Mới

```
Deployment Controller đang watch API Server:
  "GET /apis/apps/v1/deployments?watch=true"

Nhận event: {type: "ADDED", object: nginx-deployment}
   │
   ▼
Deployment Controller đọc spec.replicas = 3
Tìm ReplicaSet thuộc Deployment này:
  kubectl get replicaset -l app=nginx,pod-template-hash=<hash>
→ Không có → Cần tạo ReplicaSet mới
   │
   ▼
Tạo ReplicaSet với:
  - name: nginx-deployment-<pod-template-hash>
  - replicas: 3
  - selector: app=nginx, pod-template-hash=<hash>
  - pod template: copy từ Deployment template
```

### Bước 2.2: ReplicaSet Controller Tạo Pod

```
ReplicaSet Controller nhận event: {type: "ADDED", object: nginx-rs}

Reconciliation Loop (Vòng lặp điều hoà):
  desiredReplicas = 3
  currentPods = 0 (ReplicaSet mới tạo, chưa có Pod)
  diff = 3 - 0 = 3 → Cần tạo 3 Pod

Tạo 3 Pod object trong etcd:
  - Pod 1: nginx-deployment-xxx-aaa (Pending, chưa có nodeName)
  - Pod 2: nginx-deployment-xxx-bbb (Pending, chưa có nodeName)
  - Pod 3: nginx-deployment-xxx-ccc (Pending, chưa có nodeName)
```

---

## Giai Đoạn 3 — Scheduler

### Bước 3.1: Scheduler Phát Hiện Pod Chưa Được Gán Node

```
Scheduler đang watch API Server cho Pod chưa có nodeName:
  "GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=="

Nhận event: {type: "ADDED", object: nginx-deployment-xxx-aaa (nodeName: "")}
   │
   ▼
Bắt đầu quy trình scheduling cho Pod này
```

### Bước 3.2: Filtering — Lọc Node Không Phù Hợp

```
Danh sách node hiện tại trong cluster:
  node-1: 4 CPU, 8Gi RAM còn trống, zone: us-east-1a
  node-2: 4 CPU, 4Gi RAM còn trống, zone: us-east-1b
  node-3: 2 CPU, 2Gi RAM còn trống, zone: us-east-1c

Pod yêu cầu: requests.cpu=250m, requests.memory=256Mi

Filter: NodeResourcesFit
  → node-1: 4000m - 250m = 3750m CPU còn lại → OK
  → node-2: 4000m - 250m = 3750m CPU còn lại → OK
  → node-3: 2000m - 250m = 1750m CPU còn lại → OK
  → Tất cả đều pass

Filter: TaintToleration
  → node-1: taint dedicated=db:NoSchedule → Pod không có toleration → LOẠI
  → node-2: không có taint → OK
  → node-3: không có taint → OK

Còn lại sau filtering: node-2, node-3
```

### Bước 3.3: Scoring — Tính Điểm Node

```
Score: LeastAllocated (ưu tiên node ít tải hơn):
  node-2: 4Gi / 8Gi = 50% memory used → điểm: 50
  node-3: 2Gi / 4Gi = 50% memory used → điểm: 50

Score: InterPodAffinity / TopologySpread:
  Pod 1 và Pod 2 đã chạy trên node-2 (zone us-east-1b)
  → Ưu tiên node-3 (zone us-east-1c) để phân tán → điểm cộng thêm

Kết quả: node-3 > node-2

Gán nodeName cho Pod: nginx-deployment-xxx-aaa.spec.nodeName = "node-3"
   → API Server cập nhật Pod trong etcd
```

---

## Giai Đoạn 4 — kubelet và Container Runtime

### Bước 4.1: kubelet Phát Hiện Pod Được Gán Cho Node

```
kubelet trên node-3 đang watch API Server:
  "GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=node-3"

Nhận event: {type: "MODIFIED", object: nginx-deployment-xxx-aaa (nodeName: "node-3")}
   │
   ▼
kubelet bắt đầu xử lý Pod mới này
```

### Bước 4.2: Chuẩn Bị Pod Environment

```
1. Tạo Pod directory: /var/lib/kubelet/pods/<pod-uid>/

2. Resolve ConfigMap và Secret:
   → Gọi API Server lấy ConfigMap "app-config"
   → Gọi API Server lấy Secret "db-credentials"
   → Ghi file vào /var/lib/kubelet/pods/<pod-uid>/volumes/

3. Gọi CNI plugin để chuẩn bị network namespace:
   $ /opt/cni/bin/calico ADD < {pod network config}
   → Tạo veth pair
   → Cấp IP: 10.244.3.15/24
   → Cập nhật routing table
```

### Bước 4.3: Tạo Container Qua CRI

```
kubelet gọi containerd qua CRI (gRPC):

Bước A: Tạo Pod Sandbox (Pause Container)
  kubelet → RunPodSandbox(podSandboxConfig) → containerd
  containerd:
    → Tạo container "pause" với image gcr.io/pause:3.9
    → Container pause giữ network/IPC namespace
    → Trả về sandboxID

Bước B: Chạy Init Container (nếu có)
  kubelet → PullImage(wait-for-db:busybox) → containerd
  containerd → kéo image từ registry (nếu chưa có)
  kubelet → CreateContainer(wait-for-db, sandboxID) → containerd
  kubelet → StartContainer(containerID) → containerd
  kubelet → chờ container exit 0 (success)

Bước C: Chạy Container Chính
  kubelet → PullImage(nginx:1.25) → containerd
  containerd → kéo image từ registry
  kubelet → CreateContainer(nginx, sandboxID) → containerd
  kubelet → StartContainer(containerID) → containerd
```

### Bước 4.4: Health Probe (Kiểm Tra Sức Khoẻ)

```
Startup Probe (nếu cấu hình):
  kubelet gọi HTTP GET /startup mỗi 5 giây
  → Thành công: chuyển sang Liveness/Readiness Probe
  → Thất bại quá failureThreshold lần: restart container

Liveness Probe:
  kubelet gọi HTTP GET /healthz mỗi 10 giây
  → Thành công: container healthy
  → Thất bại quá failureThreshold lần: kill container → restart

Readiness Probe:
  kubelet gọi HTTP GET /ready mỗi 5 giây
  → Thành công: đánh dấu container Ready → Pod được thêm vào Service Endpoints
  → Thất bại: đánh dấu container NotReady → Xoá khỏi Service Endpoints
             (container không bị restart, chỉ không nhận traffic)
```

### Bước 4.5: Cập Nhật Trạng Thái

```
kubelet cập nhật Pod status lên API Server:
  status.phase: Running
  status.conditions[0]: {type: Ready, status: True}
  status.containerStatuses[0]: {
    name: nginx,
    ready: true,
    restartCount: 0,
    state: {running: {startedAt: "2026-05-10T..."}}
  }
  status.podIP: 10.244.3.15
```

---

## Giai Đoạn 5 — Networking và Service

### Bước 5.1: Endpoints Controller Cập Nhật

```
Endpoints Controller watch Pod status:
  Nhận event: Pod nginx-deployment-xxx-aaa status.conditions.Ready = True

Tìm Service nào có selector khớp với Pod này:
  Service "nginx-service" có selector: app=nginx → khớp!

Cập nhật Endpoints object:
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: nginx-service
  subsets:
    - addresses:
        - ip: 10.244.3.15     ← Pod mới vừa được thêm vào
          nodeName: node-3
          targetRef:
            kind: Pod
            name: nginx-deployment-xxx-aaa
      ports:
        - port: 8080
```

### Bước 5.2: kube-proxy Cập Nhật iptables

```
kube-proxy trên MỌI node nhận event: Endpoints "nginx-service" đã thay đổi

kube-proxy cập nhật iptables rules trên mỗi node:
  Trước: ClusterIP 10.96.1.100:80 → [Pod-A, Pod-B]
  Sau:   ClusterIP 10.96.1.100:80 → [Pod-A, Pod-B, Pod-mới (10.244.3.15:8080)]

Traffic đến Service nginx-service:80 bây giờ sẽ được load balance
đến 3 Pod backend, bao gồm Pod vừa ready.
```

---

## Sơ Đồ Đầy Đủ

```
kubectl apply -f deployment.yaml
│
├─[1]─► API Server ──────────────────────────────────────────────────────────────┐
│        │ Authentication → Authorization → Admission Control → Lưu etcd         │
│        │ Trả 201 Created cho kubectl                                            │
│        │                                                                        │
├─[2]─► Controller Manager (watch API Server)                                    │
│        │                                                                        │
│        ├── Deployment Controller                                                │
│        │     → Tạo ReplicaSet                                                  │
│        │                                                                        │
│        └── ReplicaSet Controller                                                │
│              → Tạo 3 Pod (Pending, nodeName="")                                │
│                                                                                 │
├─[3]─► Scheduler (watch Pod chưa có nodeName)                                  │
│        → Filter node → Score → Gán nodeName cho mỗi Pod                        │
│                                                                                 │
├─[4]─► kubelet trên node được chọn (watch Pod có nodeName = node này)           │
│        → Chuẩn bị: volume, network (CNI), secret                               │
│        → Tạo sandbox → init container → container chính                        │
│        → Health probe: Startup → Liveness → Readiness                          │
│        → Cập nhật Pod status: Running, Ready=True                              │
│                                                                                 │
├─[5]─► Endpoints Controller (watch Pod Ready)                                   │
│        → Cập nhật Endpoints của Service                                         │
│                                                                                 │
└─[6]─► kube-proxy trên mọi node (watch Endpoints thay đổi)
         → Cập nhật iptables/IPVS rules
         → Pod nhận được traffic! ✓
```

**Thời gian điển hình cho từng giai đoạn:**

| Giai Đoạn             | Thời Gian     | Ghi Chú                                   |
| --------------------- | ------------- | ----------------------------------------- |
| API Server xử lý      | < 1 giây      | Authentication, validation, lưu etcd      |
| Controller Manager    | < 1 giây      | Tạo ReplicaSet và Pod object              |
| Scheduler             | < 1 giây      | Chọn node và gán                          |
| kubelet — kéo image   | 5–60 giây     | Phụ thuộc kích thước image và network     |
| kubelet — khởi động   | 1–10 giây     | Phụ thuộc ứng dụng                        |
| Health probe          | 5–30 giây     | Phụ thuộc initialDelaySeconds             |
| Endpoints cập nhật    | 1–3 giây      | Bao gồm iptables propagation              |
| **Tổng cộng**         | **~15–60s**   | Image đã cache: ~5–10 giây                |

---

## Luồng Xoá Deployment

Khi chạy `kubectl delete deployment nginx`:

```
1. kubectl gửi DELETE /apis/apps/v1/namespaces/default/deployments/nginx

2. API Server đặt DeletionTimestamp lên Deployment
   → Không xoá ngay — bắt đầu Graceful Deletion

3. Deployment Controller thấy DeletionTimestamp → xoá ReplicaSet
4. ReplicaSet Controller thấy DeletionTimestamp → xoá Pod

5. Xoá Pod — kubelet nhận sự kiện Pod bị xoá:
   a. Gửi SIGTERM đến container (signal yêu cầu dừng)
   b. Chờ terminationGracePeriodSeconds (mặc định: 30 giây)
   c. Nếu container chưa dừng → gửi SIGKILL (buộc dừng)
   d. Gọi CNI plugin để giải phóng network namespace
   e. Unmount volume
   f. Xoá container và sandbox

6. Endpoints Controller nhận Pod bị xoá → xoá IP khỏi Endpoints
7. kube-proxy cập nhật iptables → không route traffic đến Pod đã xoá

8. Sau khi tất cả finalizer hoàn thành → Pod, ReplicaSet, Deployment bị xoá khỏi etcd
```

**Vấn đề thường gặp trong Graceful Deletion:**

```
Tình huống: Traffic vẫn đến Pod trong vài giây sau khi SIGTERM
Lý do: kube-proxy propagation chậm hơn terminationGracePeriodSeconds

Giải pháp: Thêm preStop hook để delay shutdown
spec:
  containers:
    - lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]   # Chờ 5 giây để iptables cập nhật trên tất cả node
```

---

## Luồng Rolling Update

Khi chạy `kubectl set image deployment/nginx nginx=nginx:1.26`:

```
1. Deployment spec.template.spec.containers[0].image = nginx:1.26
   → Pod template hash thay đổi → trigger rolling update

2. Deployment Controller tạo ReplicaSet MỚI (nginx-dep-<new-hash>):
   replicas: 0 (chưa scale up ngay)

3. Rolling update bắt đầu dựa trên strategy:
   maxUnavailable: 1 → tối đa 1 Pod có thể Unavailable
   maxSurge: 1       → tối đa 1 Pod dư ra

Timeline với 3 replica ban đầu:
   Bước 1: Scale up ReplicaSet mới lên 1 (tổng: 4 Pod, dư 1 — bằng maxSurge)
   Bước 2: Scale down ReplicaSet cũ xuống 2 (tổng: 3 Pod — không vượt maxUnavailable)
   Bước 3: Scale up mới lên 2 → Scale down cũ xuống 1
   Bước 4: Scale up mới lên 3 → Scale down cũ xuống 0

4. Rollout hoàn thành:
   ReplicaSet mới: 3 Pod đang chạy image nginx:1.26
   ReplicaSet cũ: 0 Pod (vẫn tồn tại để rollback)

5. Rollout dừng nếu Pod mới không Ready trong thời gian progressDeadlineSeconds
   → Deployment status: ProgressDeadlineExceeded
   → Thực hiện rollback tự động (nếu cấu hình)
```

---

## Debug Mỗi Giai Đoạn

### Stuck ở "Pending" — Scheduler Không Gán Node

```bash
# Xem lý do Pod Pending
kubectl describe pod <pod-name>
# Tìm section "Events:" ở dưới cùng
# Thông báo thường gặp:
#   "0/3 nodes are available: 3 Insufficient cpu."
#   "0/3 nodes are available: 3 node(s) had taint {dedicated: gpu}..."
#   "0/3 nodes are available: 3 node(s) didn't match Pod's node affinity"

# Kiểm tra resource còn lại trên các node
kubectl describe nodes | grep -A 5 "Allocated resources"

# Xem taint trên node
kubectl get nodes -o json | jq '.items[].spec.taints'
```

### Stuck ở "ContainerCreating" — kubelet Đang Xử Lý

```bash
# Xem chi tiết Events
kubectl describe pod <pod-name>
# Thông báo thường gặp:
#   "Failed to pull image: ... unauthorized"    → Lỗi kéo image
#   "MountVolume.SetUp failed: PVC not bound"   → PVC chưa được bind
#   "Failed to create pod sandbox: ..."          → Lỗi CNI network

# Xem log kubelet trên node
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'
ssh <node-ip> journalctl -u kubelet -n 100 --no-pager
```

### Pod "Running" Nhưng Không Ready

```bash
# Pod Running nhưng Readiness probe fail
kubectl describe pod <pod-name>
# Events: "Readiness probe failed: HTTP probe failed..."

# Xem log container
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Log của container trước khi restart

# Exec vào container để debug trực tiếp
kubectl exec -it <pod-name> -- sh
# Thử gọi endpoint readiness từ bên trong
wget -qO- http://localhost:8080/ready
```

### Service Không Route Traffic Đến Pod

```bash
# Kiểm tra Endpoints
kubectl get endpoints <service-name>
# Nếu "<none>" hoặc thiếu Pod → selector không khớp

# Kiểm tra selector
kubectl describe service <service-name>
kubectl get pods --show-labels | grep <app-name>

# Test kết nối từ pod khác
kubectl run test --image=busybox --rm -it -- wget -qO- http://<service-name>:<port>

# Debug iptables rules
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-<xxx>
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Mô tả đầy đủ luồng từ `kubectl apply` đến Pod chạy

**Trả lời chuẩn (5 bước):**

> "Khi tôi chạy kubectl apply, đầu tiên kubectl gửi HTTP request đến API Server. API Server xác thực (authentication), phân quyền (RBAC authorization), và qua Admission Controller (validate, mutate). Sau đó lưu Deployment vào etcd.
>
> Deployment Controller watch API Server, phát hiện Deployment mới và tạo ReplicaSet. ReplicaSet Controller phát hiện thiếu Pod và tạo Pod object với nodeName rỗng.
>
> Scheduler watch Pod chưa có node, chạy Filtering (loại node không đủ tài nguyên, taint...) rồi Scoring (chấm điểm), gán node phù hợp cho Pod.
>
> kubelet trên node được chọn watch API Server, thấy Pod mới gán cho mình: chuẩn bị volume, gọi CNI plugin cấp IP, rồi gọi container runtime (containerd) qua CRI để kéo image và khởi động container. kubelet chạy health probe và báo cáo Pod Ready.
>
> Cuối cùng, Endpoints Controller cập nhật Endpoints của Service, kube-proxy trên mọi node cập nhật iptables. Traffic từ Service bắt đầu đến được Pod mới."

### Q2: Điều gì xảy ra khi một Pod crash?

**Trả lời:**
> "Khi container trong Pod crash (exit code ≠ 0), kubelet phát hiện ngay vì nó liên tục theo dõi container. Kubelet khởi động lại container theo restartPolicy (Always hoặc OnFailure). Lần restart đầu tiên ngay lập tức, nhưng sau đó có **exponential backoff** (thời gian chờ tăng dần theo luỹ thừa): 10s, 20s, 40s, 80s, tối đa 5 phút. Đây là trạng thái CrashLoopBackOff.
>
> Trong thời gian container không Ready: Readiness probe fail → kubelet cập nhật Pod condition → Endpoints Controller xoá IP Pod khỏi Service Endpoints → kube-proxy cập nhật iptables → Pod không nhận traffic mới. Khi container restart và Readiness probe pass → tự động thêm lại vào Endpoints."

### Q3: Tại sao có delay khi xoá Pod và traffic vẫn đến Pod đó?

**Trả lời:**
> "Đây là race condition phổ biến: khi Pod bị xoá, kubelet gửi SIGTERM và bắt đầu chờ graceful shutdown. Đồng thời, Endpoints Controller phát hiện Pod không còn Ready và cập nhật Endpoints — nhưng việc propagate thay đổi iptables đến tất cả node trong cluster mất 1–3 giây. Trong thời gian này, traffic vẫn có thể đến Pod đang trong quá trình shutdown.
>
> Giải pháp: dùng **preStop hook** với `sleep 5` để Pod chờ 5 giây trước khi xử lý SIGTERM — đủ thời gian để iptables cập nhật trên tất cả node. Đây là pattern tiêu chuẩn cho zero-downtime deployment."

### Q4: maxUnavailable và maxSurge trong RollingUpdate có nghĩa gì?

**Trả lời:**
> "maxUnavailable là số Pod **tối đa** có thể ở trạng thái Unavailable (không Ready) trong quá trình rolling update — đảm bảo service capacity tối thiểu. maxSurge là số Pod **dư ra** tối đa so với desired replicas — kiểm soát tài nguyên bổ sung cần thiết trong quá trình update.
>
> Ví dụ: 3 replicas, maxUnavailable=1, maxSurge=1: trong quá trình update có tối đa 4 Pod (3+1) và tối thiểu 2 Pod Ready (3-1). Nếu set maxUnavailable=0, maxSurge=1: zero-downtime nhưng cần thêm tài nguyên. Nếu set maxUnavailable=1, maxSurge=0: tiết kiệm tài nguyên nhưng giảm 1/3 capacity trong quá trình update."

---

**Xem lại:** [README.md](./README.md) để có cái nhìn tổng thể về kiến trúc Kubernetes.
