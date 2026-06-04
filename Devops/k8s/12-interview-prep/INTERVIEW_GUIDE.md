# Top 30 Câu Hỏi Phỏng Vấn Kubernetes — Hướng Dẫn Trả Lời

> Tổng hợp 30 câu hỏi phổ biến nhất trong phỏng vấn K8s, kèm đáp án chi tiết, ví dụ thực tế và điểm mấu chốt cần nhấn mạnh.

## Mục Lục

1. [Kiến Trúc (Câu 1–7)](#1-kiến-trúc)
2. [Workload (Câu 8–13)](#2-workload)
3. [Networking (Câu 14–18)](#3-networking)
4. [Storage (Câu 19–21)](#4-storage)
5. [Bảo Mật & RBAC (Câu 22–25)](#5-bảo-mật--rbac)
6. [Scaling & Performance (Câu 26–28)](#6-scaling--performance)
7. [Vận Hành & Sự Cố (Câu 29–30)](#7-vận-hành--sự-cố)

---

## 1. Kiến Trúc

### Câu 1: Mô tả kiến trúc Kubernetes và vai trò của từng thành phần

**Câu trả lời mẫu:**

Kubernetes có 2 loại node chính:

**Control Plane (Mặt Điều Khiển)** — bộ não của cluster:
- **API Server** — cổng vào duy nhất cho mọi thao tác, nhận và xác thực request
- **etcd** — cơ sở dữ liệu key-value phân tán lưu toàn bộ trạng thái cluster
- **Scheduler** — chọn node phù hợp để đặt Pod dựa trên tài nguyên và constraint
- **Controller Manager** — vòng lặp điều khiển liên tục đảm bảo trạng thái thực khớp trạng thái mong muốn

**Worker Node (Nút Xử Lý)** — nơi container thực sự chạy:
- **kubelet** — agent trên mỗi node, nhận PodSpec và đảm bảo container chạy đúng
- **kube-proxy** — duy trì iptables rule để định tuyến traffic đến đúng Pod
- **Container Runtime (Thời Gian Chạy Container)** — containerd hoặc CRI-O, thực sự chạy container

> **Điểm cần nhấn mạnh:** etcd là điểm quan trọng nhất cần backup — mất etcd là mất toàn bộ trạng thái cluster.

---

### Câu 2: Luồng xử lý khi chạy `kubectl apply -f deployment.yaml` là gì?

**Câu trả lời mẫu:**

```
1. kubectl đọc file YAML, gửi HTTP PUT/POST đến API Server
2. API Server xác thực:
   - Authentication (Xác Thực): kiểm tra certificate hoặc token
   - Authorization (Phân Quyền): kiểm tra RBAC — user có quyền tạo Deployment?
   - Admission Control (Kiểm Soát Tiếp Nhận): Webhook và PSA kiểm tra policy
3. API Server ghi object vào etcd
4. Deployment Controller (trong Controller Manager) phát hiện Deployment mới
5. Controller tạo ReplicaSet với số replica mong muốn
6. ReplicaSet Controller tạo Pod object trong etcd
7. Scheduler phát hiện Pod chưa có node, chọn node phù hợp, ghi vào etcd
8. kubelet trên node được chọn phát hiện Pod mới, gọi container runtime
9. Container Runtime pull image, tạo container
10. kubelet cập nhật trạng thái Pod về etcd qua API Server
11. kube-proxy cập nhật iptables rule nếu có Service liên quan
```

> **Mấu chốt:** Mọi thứ đều đi qua API Server. Không có component nào giao tiếp trực tiếp với nhau — tất cả đọc/ghi từ etcd qua API Server.

---

### Câu 3: etcd là gì và tại sao quan trọng?

**Câu trả lời mẫu:**

etcd là **distributed key-value store (kho lưu trữ key-value phân tán)** dùng thuật toán đồng thuận Raft để đảm bảo tính nhất quán trên nhiều node.

**Kubernetes dùng etcd để lưu:**
- Toàn bộ object K8s: Pod, Deployment, Service, ConfigMap, Secret, ...
- Trạng thái cluster hiện tại
- Config map và certificate

**Tại sao quan trọng:**
- Mất etcd = mất toàn bộ trạng thái cluster
- K8s không lưu state ở đâu khác ngoài etcd
- Backup etcd định kỳ là bắt buộc trong production

**Best practice:**
```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

### Câu 4: Scheduler quyết định đặt Pod lên node nào?

**Câu trả lời mẫu:**

Scheduler dùng **2 bước**:

**Bước 1 — Filtering (Lọc):** loại bỏ các node không thể chạy Pod
- Node có đủ CPU và memory?
- Node có taint phù hợp với toleration của Pod?
- NodeSelector hoặc nodeAffinity có khớp không?
- PVC (PersistentVolumeClaim) có thể mount được từ node đó?

**Bước 2 — Scoring (Tính Điểm):** xếp hạng node còn lại
- Node nào có nhiều tài nguyên trống nhất? (LeastRequestedPriority)
- Node nào cân bằng tài nguyên tốt nhất? (BalancedResourceAllocation)
- Pod Affinity/Anti-Affinity — đặt gần hay xa Pod khác?

Node có điểm cao nhất được chọn.

**Ví dụ YAML:**
```yaml
spec:
  affinity:
    podAntiAffinity:  # Không đặt 2 Pod cùng loại trên cùng node
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: kubernetes.io/hostname
```

---

### Câu 5: Namespace (Không Gian Tên) dùng để làm gì?

**Câu trả lời mẫu:**

Namespace cung cấp **phân vùng logic** trong cluster, cho phép:

1. **Cô lập tài nguyên** — dev, staging, production chạy cùng cluster
2. **Phân quyền RBAC** — team A không thể truy cập resource của team B
3. **Giới hạn tài nguyên** — dùng ResourceQuota cho từng namespace
4. **Network isolation** — kết hợp NetworkPolicy để cô lập lưu lượng

**Không phải isolation hoàn toàn:**
- Node vẫn dùng chung
- Network mặc định vẫn thông suốt (phải cấu hình NetworkPolicy)
- ClusterRole vượt qua namespace

```bash
# Tạo namespace
kubectl create namespace production

# ResourceQuota cho namespace
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
EOF
```

---

### Câu 6: Pod khác gì với Container?

**Câu trả lời mẫu:**

**Container** là đơn vị runtime — một process chạy trong môi trường cô lập (namespace, cgroup).

**Pod** là đơn vị triển khai nhỏ nhất trong K8s, có thể chứa **một hoặc nhiều container** dùng chung:
- **Network namespace** — cùng IP, cùng port space
- **Storage volumes** — cùng truy cập volume được mount
- **Linux cgroup** — giới hạn resource chung
- **Lifecycle** — cùng khởi động và tắt

**Khi nào dùng multi-container Pod?**

| Pattern | Ví Dụ | Mục Đích |
|---------|-------|----------|
| Sidecar | Log shipper cạnh app | Augment chức năng chính |
| Adapter | Prometheus exporter | Chuyển đổi format |
| Ambassador | Service mesh proxy | Proxy traffic |
| Init Container | Wait for DB ready | Tiền xử lý trước khi app chạy |

---

### Câu 7: Controller là gì? Nêu các controller quan trọng

**Câu trả lời mẫu:**

Controller là **vòng lặp điều khiển (control loop)** liên tục so sánh *desired state* (trạng thái mong muốn trong etcd) với *current state* (trạng thái thực tế) và thực hiện hành động để thu hẹp khoảng cách.

**Các controller quan trọng:**

| Controller | Nhiệm Vụ |
|-----------|---------|
| Deployment Controller | Tạo/quản lý ReplicaSet |
| ReplicaSet Controller | Đảm bảo đúng số lượng Pod |
| Node Controller | Phát hiện node down, evict Pod |
| Job Controller | Chạy Pod đến khi hoàn thành |
| Endpoint Controller | Cập nhật danh sách IP Pod vào Service |
| Namespace Controller | Xoá resource khi namespace bị xoá |

```
Vòng lặp điều khiển (pseudocode):
while true:
    desired = etcd.get(resourceSpec)
    current = cluster.observe()
    if desired != current:
        cluster.reconcile(desired)
    sleep(interval)
```

---

## 2. Workload

### Câu 8: Deployment, ReplicaSet và Pod có quan hệ thế nào?

**Câu trả lời mẫu:**

Đây là **quan hệ phân cấp một chiều**:

```
Deployment (quản lý chiến lược update)
    └── ReplicaSet v2 (phiên bản mới, replicas=3)
    └── ReplicaSet v1 (phiên bản cũ, replicas=0 sau khi update xong)
            └── Pod (x3)
```

- **Deployment** quản lý rolling update strategy — biết cách tạo ReplicaSet mới và scale xuống cái cũ
- **ReplicaSet** đảm bảo đúng số Pod đang chạy — không biết gì về update
- **Pod** là đơn vị thực thi — ReplicaSet xoá Pod cũ, Scheduler khởi tạo Pod mới

**Thực tế:** Bạn hiếm khi tạo ReplicaSet trực tiếp — hãy dùng Deployment.

---

### Câu 9: Cấu hình Rolling Update (Cập Nhật Cuốn Chiếu) không có downtime như thế nào?

**Câu trả lời mẫu:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Tối đa thêm 1 Pod khi update (tổng 4 Pod)
      maxUnavailable: 0    # Không Pod nào được phép down trong khi update
  template:
    spec:
      containers:
      - name: web
        image: my-app:v2
        readinessProbe:    # Bắt buộc! Traffic chỉ chuyển sang Pod mới khi ready
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Các thành phần đảm bảo zero-downtime:**
1. `maxUnavailable: 0` — luôn giữ đủ số Pod available
2. `maxSurge: 1` — tạo Pod mới trước khi xoá Pod cũ
3. `readinessProbe` — traffic chỉ chuyển sang Pod mới khi readinessProbe thành công
4. `preStop hook` — cho Pod cũ thời gian xử lý request đang bay

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]  # Chờ LB drain connections
```

---

### Câu 10: StatefulSet khác Deployment thế nào? Khi nào dùng?

**Câu trả lời mẫu:**

| Tiêu Chí | Deployment | StatefulSet |
|----------|-----------|-------------|
| Pod Identity | Pod name ngẫu nhiên (pod-xyz) | Pod name có thứ tự (pod-0, pod-1) |
| Network Identity | IP thay đổi | DNS hostname ổn định (pod-0.service) |
| Storage | Volume dùng chung hoặc ephemeral | Mỗi Pod có PVC riêng (VolumeClaimTemplate) |
| Thứ tự khởi động | Song song | Tuần tự: pod-0 → pod-1 → pod-2 |
| Thứ tự xoá | Song song | Ngược lại: pod-2 → pod-1 → pod-0 |

**Dùng StatefulSet khi:**
- Database: PostgreSQL, MySQL, MongoDB
- Message queue: Kafka, RabbitMQ
- Distributed cache: Redis Cluster
- Bất kỳ workload nào cần **stable identity** hoặc **dedicated storage**

**Ví dụ VolumeClaimTemplate:**
```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```
Mỗi Pod (db-0, db-1, db-2) sẽ có PVC riêng: `data-db-0`, `data-db-1`, `data-db-2`.

---

### Câu 11: Liveness Probe và Readiness Probe khác nhau thế nào?

**Câu trả lời mẫu:**

| | Liveness Probe | Readiness Probe | Startup Probe |
|-|---------------|-----------------|---------------|
| **Mục đích** | Pod còn sống không? | Pod sẵn sàng nhận traffic? | Pod đã khởi động xong chưa? |
| **Khi thất bại** | K8s restart container | Xóa Pod khỏi Service endpoint | Không chạy Liveness/Readiness |
| **Dùng khi** | App có thể deadlock | App cần warmup time | App khởi động chậm |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3    # Restart sau 3 lần thất bại liên tiếp

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3    # Tạm thời không nhận traffic (không restart)
```

> **Điểm quan trọng:** Không có readinessProbe thì rolling update không an toàn — traffic sẽ được gửi đến Pod mới ngay cả khi app chưa khởi động xong.

---

### Câu 12: Init Container (Container Khởi Tạo) dùng để làm gì?

**Câu trả lời mẫu:**

Init Container chạy **hoàn toàn trước** khi container chính khởi động. Chúng chạy **tuần tự** và **phải thành công** (exit code 0) trước khi container tiếp theo chạy.

**Use case phổ biến:**
- Chờ database sẵn sàng trước khi app chạy
- Tải config từ external source (Vault, S3)
- Chạy database migration
- Clone git repository vào shared volume

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c',
    'until nc -z postgres-service 5432; do echo waiting; sleep 2; done']
- name: run-migrations
  image: my-app:v2
  command: ['./migrate', 'up']
  env:
  - name: DB_URL
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: url
containers:
- name: web-app
  image: my-app:v2
```

---

### Câu 13: DaemonSet dùng khi nào?

**Câu trả lời mẫu:**

DaemonSet đảm bảo **mỗi node chạy đúng một bản sao** của Pod. Khi node mới join cluster, Pod tự động được tạo trên node đó.

**Trường hợp dùng:**
- **Log collection:** Fluentd, Filebeat thu thập log từ mọi node
- **Monitoring agent:** Prometheus Node Exporter, Datadog agent
- **Network plugin:** Calico, Flannel, Cilium cần chạy trên mọi node
- **Storage daemon:** Ceph, GlusterFS agent
- **Security scanner:** Falco runtime security

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    spec:
      tolerations:      # Chạy cả trên master node
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100
```

---

## 3. Networking

### Câu 14: Phân biệt ClusterIP, NodePort, LoadBalancer, ExternalName

**Câu trả lời mẫu:**

| Type | Truy Cập Từ | Khi Nào Dùng |
|------|------------|--------------|
| **ClusterIP** | Trong cluster (mặc định) | Service nội bộ, microservice giao tiếp nhau |
| **NodePort** | Ngoài cluster qua `<NodeIP>:<NodePort>` | Lab, testing (30000–32767) |
| **LoadBalancer** | Internet qua cloud LB | Production, expose service ra ngoài |
| **ExternalName** | Trong cluster | Map service name tới CNAME ngoài cluster |

**ClusterIP — ví dụ điển hình:**
```yaml
# API gateway gọi user-service bằng http://user-service:8080
# kube-proxy định tuyến đến đúng Pod qua iptables
spec:
  type: ClusterIP
  ports:
  - port: 8080       # Port cluster nội bộ
    targetPort: 8080 # Port container
```

**LoadBalancer — cơ chế:**
1. K8s yêu cầu cloud provider (AWS/GCP/Azure) tạo Load Balancer
2. Cloud LB nhận traffic từ internet → NodePort
3. NodePort → ClusterIP → Pod

> **Mẹo phỏng vấn:** Giải thích tại sao ít dùng NodePort trong production (expose port trên mọi node không an toàn, phải mở firewall rule).

---

### Câu 15: Ingress hoạt động thế nào?

**Câu trả lời mẫu:**

**Ingress** là rule routing HTTP/HTTPS, **Ingress Controller** là implementation thực thi rule đó.

```
Internet → Load Balancer (L4)
              ↓
        Ingress Controller Pod (NGINX/Traefik/AWS ALB)
              ↓ (đọc Ingress rules, routing theo host/path)
        Service A (api.example.com/users)
        Service B (api.example.com/orders)
        Service C (app.example.com)
```

**Ví dụ Ingress rule:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret     # Certificate lưu trong Secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

**Ingress Controller phổ biến:** NGINX, Traefik, AWS ALB Ingress Controller, Kong

---

### Câu 16: CoreDNS giải quyết tên service như thế nào?

**Câu trả lời mẫu:**

CoreDNS là **DNS server** chạy trong cluster, tự động đăng ký DNS record cho mọi Service.

**Format DNS chuẩn:**
```
<service-name>.<namespace>.svc.<cluster-domain>
```

Ví dụ: `user-service.production.svc.cluster.local`

**Cách resolve từ trong Pod:**
```
# Pod trong cùng namespace — resolve ngắn
curl http://user-service

# Pod khác namespace — cần full name
curl http://user-service.production

# Full DNS name — luôn hoạt động
curl http://user-service.production.svc.cluster.local
```

**Cơ chế:**
1. Pod được cấu hình `nameserver: ClusterIP-of-CoreDNS` trong `/etc/resolv.conf`
2. DNS query → CoreDNS Pod
3. CoreDNS tra cứu etcd → trả về ClusterIP của Service
4. Pod gửi traffic đến ClusterIP → kube-proxy phân phối đến Pod đích

---

### Câu 17: NetworkPolicy (Chính Sách Mạng) hoạt động thế nào?

**Câu trả lời mẫu:**

NetworkPolicy kiểm soát traffic **ingress (vào)** và **egress (ra)** của Pod dựa trên label selector.

**Mặc định:** Tất cả Pod có thể giao tiếp với nhau (không có NetworkPolicy = allow all).

**Deny all ingress, chỉ allow từ frontend:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server      # Policy áp dụng cho Pod có label này
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend    # Chỉ frontend Pod mới được kết nối vào
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres    # API chỉ được kết nối ra postgres
    ports:
    - protocol: TCP
      port: 5432
  - to: []                 # DNS lookup (cần allow UDP 53)
    ports:
    - protocol: UDP
      port: 53
```

> **Lưu ý:** NetworkPolicy cần CNI plugin hỗ trợ (Calico, Cilium, Weave Net). Flannel thuần không hỗ trợ NetworkPolicy.

---

### Câu 18: Khi Service không kết nối được, bạn debug thế nào?

**Câu trả lời mẫu:**

```bash
# Bước 1: Kiểm tra Pod có đang chạy không
kubectl get pods -l app=my-service

# Bước 2: Kiểm tra Service có Endpoint không
kubectl get endpoints my-service
# Nếu ENDPOINTS là <none> → label không khớp hoặc Pod không ready

# Bước 3: Kiểm tra label Pod có khớp selector của Service không
kubectl get pods --show-labels
kubectl describe service my-service  # Xem selector

# Bước 4: Test DNS từ trong cluster
kubectl run debug --image=busybox -it --rm -- nslookup my-service

# Bước 5: Test kết nối trực tiếp đến Pod (bypass Service)
kubectl run debug --image=busybox -it --rm -- wget -O- <pod-ip>:8080

# Bước 6: Test qua Service ClusterIP
kubectl run debug --image=busybox -it --rm -- wget -O- <cluster-ip>:8080

# Bước 7: Kiểm tra NetworkPolicy
kubectl get networkpolicy -n <namespace>
```

**Nguyên nhân phổ biến:**
- Label Pod không khớp selector của Service
- Port cấu hình sai (targetPort ≠ containerPort)
- NetworkPolicy block traffic
- Pod không pass readinessProbe → không vào Endpoint list

---

## 4. Storage

### Câu 19: PV và PVC khác nhau thế nào?

**Câu trả lời mẫu:**

**Mô hình tương tự API provider–consumer:**

| | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
|-|----------------------|----------------------------|
| **Ai tạo** | Admin (hoặc dynamic provisioner) | Developer / Application |
| **Mô tả** | Tài nguyên storage thực tế (NFS, EBS, GCE PD) | Yêu cầu storage (cần 10Gi, ReadWriteOnce) |
| **Lifecycle** | Độc lập với Pod | Gắn với namespace |
| **Tương tự** | Phòng trong khách sạn | Đơn đặt phòng |

**Dynamic Provisioning (Cấp Phát Động)** với StorageClass:
```yaml
# StorageClass — Admin tạo một lần
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete  # Xoá PV khi PVC bị xoá

---
# PVC — Developer tạo
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast-ssd
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
# K8s tự động tạo PV (EBS volume) tương ứng
```

---

### Câu 20: Access Mode của PersistentVolume có những loại nào?

**Câu trả lời mẫu:**

| Access Mode | Ký Hiệu | Mô Tả | Ví Dụ |
|-------------|---------|-------|-------|
| ReadWriteOnce (RWO) | Đọc/ghi từ 1 node | Một node mount volume đọc/ghi | EBS, local disk |
| ReadOnlyMany (ROX) | Chỉ đọc từ nhiều node | Nhiều node mount volume chỉ đọc | NFS (read), ConfigMap |
| ReadWriteMany (RWX) | Đọc/ghi từ nhiều node | Nhiều node mount volume đọc/ghi | NFS, EFS, Ceph |
| ReadWriteOncePod (RWOP) | Đọc/ghi từ 1 Pod | Chỉ 1 Pod duy nhất mount | Bảo mật cao nhất |

> **Thực tế:** AWS EBS chỉ hỗ trợ RWO. Nếu cần shared storage (nhiều Pod cùng ghi), dùng AWS EFS (hỗ trợ RWX) hoặc xem xét lại kiến trúc.

---

### Câu 21: Khi nào dùng emptyDir vs hostPath vs PersistentVolume?

**Câu trả lời mẫu:**

| Volume Type | Lifecycle | Dùng Khi | Cẩn Thận |
|-------------|-----------|---------|----------|
| **emptyDir** | Theo Pod (xoá khi Pod xoá) | Chia sẻ data giữa container trong Pod, cache tạm | Mất data khi Pod restart |
| **hostPath** | Theo Node (tồn tại khi Pod xoá) | DaemonSet cần đọc file system host, log agent | Gắn chặt vào node, security risk |
| **PersistentVolume** | Độc lập | Database, file upload, bất kỳ data cần bền vững | Cần provisioner, chi phí storage |

```yaml
# emptyDir — share data giữa sidecar và app container
volumes:
- name: shared-logs
  emptyDir: {}

# emptyDir với memory (RAM disk — rất nhanh nhưng giới hạn bởi node RAM)
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 256Mi
```

---

## 5. Bảo Mật & RBAC

### Câu 22: Giải thích RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò)

**Câu trả lời mẫu:**

RBAC có **4 thành phần:**

```
Role / ClusterRole     →    Định nghĩa quyền (được làm gì với resource nào)
RoleBinding / ClusterRoleBinding → Gán quyền cho Subject (ai)
Subject                →    User, Group, ServiceAccount
```

| Resource | Scope | Dùng Khi |
|----------|-------|---------|
| Role | Namespace | Quyền trong 1 namespace |
| ClusterRole | Cluster-wide | Quyền trên toàn cluster hoặc cho node/PV |
| RoleBinding | Namespace | Gán Role/ClusterRole trong namespace |
| ClusterRoleBinding | Cluster-wide | Gán ClusterRole trên toàn cluster |

**Ví dụ thực tế — Developer chỉ được xem Pod trong namespace `dev`:**
```yaml
# Role: chỉ get/list/watch pods
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding: gán cho developer
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader
  namespace: dev
subjects:
- kind: User
  name: "john@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### Câu 23: ServiceAccount (Tài Khoản Dịch Vụ) dùng để làm gì?

**Câu trả lời mẫu:**

ServiceAccount là **identity cho Pod** khi Pod cần gọi Kubernetes API hoặc cloud service.

**3 trường hợp dùng chính:**

1. **Pod gọi K8s API** (ArgoCD đọc Deployment, Prometheus scrape metrics)
2. **Pod truy cập cloud resource** (IRSA trên EKS, Workload Identity trên GKE)
3. **CI/CD pipeline** tương tác với cluster

**IRSA — IAM Roles for Service Accounts (Vai Trò IAM cho Tài Khoản Dịch Vụ) trên EKS:**
```yaml
# ServiceAccount với annotation IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/S3ReadRole

---
# Pod dùng ServiceAccount này có thể đọc S3 mà không cần AWS credentials
spec:
  serviceAccountName: s3-reader
  containers:
  - name: app
    image: my-app:v2
    # AWS SDK tự động lấy credential từ mounted token
```

> **Best practice:** Tạo ServiceAccount riêng cho mỗi workload. Không dùng `default` ServiceAccount cho app production.

---

### Câu 24: Pod Security Admission (PSA) là gì?

**Câu trả lời mẫu:**

PSA (thay thế PodSecurityPolicy từ K8s 1.25) kiểm tra Pod có tuân thủ security standard trước khi được tạo.

**3 mức độ (profile):**

| Profile | Mô Tả | Dùng Cho |
|---------|-------|---------|
| Privileged (Đặc Quyền) | Không hạn chế | System component, CNI plugin |
| Baseline (Cơ Bản) | Ngăn privilege escalation phổ biến | Workload thông thường |
| Restricted (Hạn Chế) | Best practice bảo mật nghiêm ngặt nhất | Workload xử lý data nhạy cảm |

**Cấu hình qua label namespace:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted   # Từ chối Pod không tuân thủ
    pod-security.kubernetes.io/audit: restricted      # Ghi log vi phạm
    pod-security.kubernetes.io/warn: restricted       # Cảnh báo user
```

**SecurityContext quan trọng:**
```yaml
securityContext:
  runAsNonRoot: true          # Không chạy container với root
  runAsUser: 1000             # UID cụ thể
  readOnlyRootFilesystem: true # Filesystem chỉ đọc
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]             # Bỏ tất cả Linux capability
```

---

### Câu 25: Làm thế nào để Pod truy cập AWS S3 an toàn trên EKS?

**Câu trả lời mẫu:**

**Cách đúng: IRSA (IAM Roles for Service Accounts)**

```
1. Tạo IAM Policy cho phép đọc S3 bucket
2. Tạo IAM Role, trust policy cho EKS OIDC provider
3. Tạo ServiceAccount trong K8s với annotation IAM Role ARN
4. Deployment dùng ServiceAccount đó
5. AWS SDK trong container tự động lấy credential qua mounted token
```

```bash
# Tạo IAM Role với eksctl
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=production \
  --name=s3-reader \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

**Không nên làm:**
- Hardcode AWS_ACCESS_KEY_ID và AWS_SECRET_ACCESS_KEY trong Secret → bị leak qua log, etcd
- Mount AWS credentials file qua hostPath → security risk
- Dùng instance profile của node → tất cả Pod trên node đều có quyền

---

## 6. Scaling & Performance

### Câu 26: HPA (Horizontal Pod Autoscaler) cấu hình và hoạt động thế nào?

**Câu trả lời mẫu:**

HPA theo dõi metric và điều chỉnh số replica để giữ metric gần target.

**Công thức:**
```
desiredReplicas = ceil(currentReplicas × currentMetricValue / targetMetricValue)
```

**Ví dụ cấu hình đầy đủ:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # Scale khi CPU trung bình vượt 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Chờ 60s trước khi scale up thêm
      policies:
      - type: Percent
        value: 100                       # Tối đa tăng 100% số Pod mỗi lần
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Chờ 5 phút trước khi scale down
      policies:
      - type: Pods
        value: 1                         # Chỉ giảm 1 Pod mỗi lần
        periodSeconds: 60
```

> **Điều kiện:** Pod phải có `resources.requests.cpu` định nghĩa, và metrics-server phải được cài.

---

### Câu 27: Resource Request và Limit khác nhau thế nào?

**Câu trả lời mẫu:**

```yaml
resources:
  requests:         # Scheduler dùng để chọn node
    cpu: "250m"     # Container được bảo đảm có 250m CPU
    memory: "256Mi" # Container được bảo đảm có 256Mi RAM
  limits:           # Container không được vượt quá giới hạn này
    cpu: "1000m"    # CPU bị throttle nếu vượt (không bị kill)
    memory: "512Mi" # Container bị OOMKilled (Out Of Memory) nếu vượt
```

**Quy tắc thực hành:**

| Tình Huống | Khuyến Nghị |
|-----------|-------------|
| CPU limit | Có thể bỏ (throttling an toàn hơn OOMKilled) |
| Memory limit | Nên đặt (OOMKill tốt hơn node hết RAM) |
| Request:Limit ratio | Không quá 1:4 để tránh noisy neighbor |
| Burstable workload | Request < Limit |
| Guaranteed QoS | Request = Limit |

**QoS Class (Lớp Chất Lượng Dịch Vụ):**
- **BestEffort** — không có request/limit → bị evict đầu tiên
- **Burstable** — có request hoặc request < limit
- **Guaranteed** — request = limit → được bảo vệ khi node pressure

---

### Câu 28: Cluster Autoscaler hoạt động thế nào? Khác HPA thế nào?

**Câu trả lời mẫu:**

| | HPA | Cluster Autoscaler |
|-|-----|-------------------|
| **Scale gì** | Số lượng Pod | Số lượng Node |
| **Trigger** | Metric (CPU, memory, custom) | Pod Pending do thiếu node |
| **Tốc độ** | Giây đến phút | Phút (provision node mới chậm hơn) |

**Cluster Autoscaler hoạt động:**
```
1. Phát hiện Pod Pending (không có node đủ tài nguyên)
2. Tính toán: node mới nào sẽ giải quyết được Pod Pending?
3. Tăng desired count của Auto Scaling Group (AWS) / MIG (GCP)
4. Cloud provider provision node mới (2–5 phút)
5. Node join cluster, Pod được scheduled

Scale Down:
1. Node có utilization thấp (< 50% default) trong 10 phút
2. Kiểm tra: Pod trên node có thể chạy trên node khác không?
3. Drain node (evict Pod), xoá node
```

**Phối hợp HPA + Cluster Autoscaler:**
```
Traffic tăng
  → HPA tạo thêm Pod
    → Pod Pending (cluster đầy)
      → Cluster Autoscaler thêm node
        → Pod được schedule
```

---

## 7. Vận Hành & Sự Cố

### Câu 29: Pod ở trạng thái CrashLoopBackOff — bạn debug thế nào?

**Câu trả lời mẫu:**

**Quy trình debug từng bước:**

```bash
# Bước 1: Xem thông tin trạng thái Pod
kubectl get pods
# NAME         READY   STATUS             RESTARTS   AGE
# my-app-xyz   0/1     CrashLoopBackOff   5          3m

# Bước 2: Xem event và lý do restart
kubectl describe pod my-app-xyz
# Tìm phần Events: và Last State: để biết exit code

# Bước 3: Xem log của lần chạy hiện tại
kubectl logs my-app-xyz

# Bước 4: Xem log của lần chạy TRƯỚC (rất quan trọng!)
kubectl logs my-app-xyz --previous

# Bước 5: Kiểm tra exit code
# Exit code 1: Lỗi application
# Exit code 137: OOMKilled (Out Of Memory)
# Exit code 139: Segmentation fault
# Exit code 143: SIGTERM (bị kill)
```

**Nguyên nhân phổ biến:**
| Nguyên Nhân | Dấu Hiệu | Giải Pháp |
|------------|---------|-----------|
| Lỗi startup | Exit code 1, log có exception | Sửa config, env var |
| OOMKilled | Exit code 137 | Tăng memory limit |
| Liveness probe fail | Restart định kỳ không có crash | Tăng threshold, sửa endpoint |
| Missing secret/configmap | CrashLoop ngay lập tức | Kiểm tra volume mount |
| DB connection fail | Log có connection refused | Kiểm tra Service, Secret |

---

### Câu 30: Bạn thiết kế deploy zero-downtime trên Kubernetes thế nào?

**Câu trả lời mẫu:**

Zero-downtime deployment cần **nhiều lớp bảo vệ đồng thời**:

**Lớp 1 — Deployment Strategy:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # Tạo 25% Pod mới trước
    maxUnavailable: 0    # Không Pod nào down trong khi update
```

**Lớp 2 — Health Check:**
```yaml
readinessProbe:          # Chỉ gửi traffic khi app sẵn sàng
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 15
  failureThreshold: 3
```

**Lớp 3 — Graceful Shutdown:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 30   # Cho Pod 30s drain connections
```

**Lớp 4 — Pod Disruption Budget (Ngân Sách Gián Đoạn Pod):**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2         # Luôn có tối thiểu 2 Pod available
  selector:
    matchLabels:
      app: web-app
```

**Lớp 5 — Multiple Replicas + Anti-Affinity:**
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web-app
        topologyKey: kubernetes.io/hostname  # Spread Pod ra nhiều node
```

> **Tổng kết:** Zero-downtime không chỉ là `maxUnavailable: 0`. Cần kết hợp: đủ replica, readiness probe đúng, graceful shutdown, PDB, và anti-affinity để tránh Single Point of Failure.

---

## Bảng Tóm Tắt Nhanh

| # | Câu Hỏi | Từ Khóa Cần Nhớ |
|---|---------|-----------------|
| 1 | Kiến trúc K8s | API Server, etcd, Scheduler, Controller Manager, kubelet |
| 2 | kubectl apply flow | Authentication → Authorization → Admission → etcd → Controller |
| 3 | etcd | Distributed KV, Raft consensus, single source of truth |
| 4 | Scheduler | Filter → Score → Bind |
| 5 | Namespace | Logical isolation, RBAC boundary, ResourceQuota |
| 6 | Pod vs Container | Shared network, storage, lifecycle |
| 7 | Controller | Control loop, desired state vs current state |
| 8 | Deployment hierarchy | Deployment → ReplicaSet → Pod |
| 9 | Rolling Update | maxSurge, maxUnavailable, readinessProbe |
| 10 | StatefulSet | Stable identity, ordered deployment, VolumeClaimTemplate |
| 11 | Probe types | Liveness=restart, Readiness=traffic, Startup=slow start |
| 12 | Init Container | Sequential, must succeed before main container |
| 13 | DaemonSet | One Pod per node, monitoring/logging agent |
| 14 | Service types | ClusterIP, NodePort, LoadBalancer, ExternalName |
| 15 | Ingress | L7 routing, host/path based, TLS termination |
| 16 | CoreDNS | service.namespace.svc.cluster.local |
| 17 | NetworkPolicy | Whitelist model, ingress/egress rules |
| 18 | Debug Service | get endpoints, label mismatch, NetworkPolicy |
| 19 | PV vs PVC | PV=resource, PVC=claim, StorageClass=dynamic |
| 20 | Access Modes | RWO, ROX, RWX, RWOP |
| 21 | Volume types | emptyDir, hostPath, PersistentVolume |
| 22 | RBAC | Role, ClusterRole, Binding, Subject |
| 23 | ServiceAccount | Pod identity, IRSA, Workload Identity |
| 24 | Pod Security | PSA profiles: Privileged, Baseline, Restricted |
| 25 | EKS + S3 | IRSA, không hardcode credentials |
| 26 | HPA | metrics-server, formula, stabilization window |
| 27 | Resource | Request=scheduling, Limit=enforcement, QoS |
| 28 | Cluster Autoscaler | Scale node khi Pod Pending, phối hợp HPA |
| 29 | CrashLoopBackOff | logs --previous, exit code, describe |
| 30 | Zero-downtime | Strategy + probe + preStop + PDB + anti-affinity |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
