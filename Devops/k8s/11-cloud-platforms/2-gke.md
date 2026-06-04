# Google GKE — Google Kubernetes Engine

> Hướng dẫn toàn diện về Google GKE (Google Kubernetes Engine — Công Cụ Kubernetes của Google): kiến trúc, Autopilot mode, Workload Identity, node pool, tích hợp GCP và vận hành production.

## Mục Lục

1. [Tổng Quan GKE](#tổng-quan-gke)
2. [Standard Mode vs Autopilot Mode](#standard-mode-vs-autopilot-mode)
3. [Kiến Trúc GKE](#kiến-trúc-gke)
4. [Node Pool — Nhóm Node](#node-pool--nhóm-node)
5. [Workload Identity — Định Danh Workload](#workload-identity--định-danh-workload)
6. [Release Channel — Kênh Phát Hành](#release-channel--kênh-phát-hành)
7. [Networking trên GKE](#networking-trên-gke)
8. [Load Balancer trên GKE](#load-balancer-trên-gke)
9. [Storage trên GKE](#storage-trên-gke)
10. [Bảo Mật GKE](#bảo-mật-gke)
11. [Tích Hợp GCP Services](#tích-hợp-gcp-services)
12. [Tối Ưu Chi Phí GKE](#tối-ưu-chi-phí-gke)
13. [Vận Hành và Upgrade](#vận-hành-và-upgrade)
14. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan GKE

Google Kubernetes Engine là managed Kubernetes service trên GCP (Google Cloud Platform). Google là công ty tạo ra Kubernetes — nên GKE thường đi đầu về tính năng mới, tích hợp tốt nhất và trải nghiệm vận hành smooth nhất.

### Tại Sao GKE?

```
1. Google tạo ra Kubernetes → GKE mature nhất
2. Autopilot mode → quản lý đơn giản nhất trong 3 managed provider
3. Cập nhật K8s phiên bản mới nhanh nhất
4. Tích hợp sâu với GCP: BigQuery, Cloud SQL, GCS, Artifact Registry
5. Control Plane miễn phí (Standard mode) — không mất $0.10/giờ như EKS
```

### Chi Phí Cơ Bản

```
Standard Mode:
  Control Plane:  Miễn phí cho 1 cluster per billing account
                  $0.10/giờ từ cluster thứ 2 trở đi
  Worker Node:    Chi phí Compute Engine VM thông thường

Autopilot Mode:
  Tính theo vCPU, memory và ephemeral storage Pod thực sự sử dụng
  Không trả phí cho node idle — tiết kiệm hơn khi tải thấp
```

---

## Standard Mode vs Autopilot Mode

### Standard Mode (Chế Độ Tiêu Chuẩn)

Bạn quản lý node pool — chọn machine type, cấu hình node, quyết định số lượng node.

```
Bạn quyết định:
- Machine type (n2-standard-4, e2-standard-8...)
- Số node per pool
- Auto-scaling min/max
- Node image (Container-Optimized OS, Ubuntu)
- Taints và labels

GKE lo:
- Control Plane (API Server, etcd, Scheduler, CM)
- Node health monitoring và auto-repair
- OS security patches (nếu bật)
```

### Autopilot Mode (Chế Độ Tự Động)

GKE quản lý toàn bộ node — bạn chỉ deploy workload. GKE tự quyết định máy nào, ở đâu, bao nhiêu.

```
Bạn quyết định:
- Workload (Pod, Deployment, StatefulSet...)
- Resource request của Pod (CPU, memory)

GKE lo:
- Chọn machine type phù hợp
- Cấp phát node khi cần
- Thu hồi node khi dư
- OS patching
- Node upgrade
- Bảo mật node (hardened OS, no SSH)
```

### So Sánh

| Tiêu Chí                     | Standard Mode               | Autopilot Mode                  |
| ---------------------------- | --------------------------- | -------------------------------- |
| Quản lý node                 | Bạn quản lý                 | GKE quản lý                      |
| Linh hoạt cấu hình node      | Cao                         | Thấp (GKE quyết định)            |
| DaemonSet                    | Hỗ trợ đầy đủ               | Chỉ DaemonSet được GKE cho phép  |
| hostPath volume              | Hỗ trợ                      | Không hỗ trợ                     |
| Privileged container         | Hỗ trợ                      | Không hỗ trợ                     |
| Chi phí khi tải thấp         | Trả cho node dù idle         | Chỉ trả cho Pod đang chạy        |
| Chi phí khi tải cao          | Có thể thấp hơn             | Có thể cao hơn                   |
| Overhead vận hành            | Trung bình                  | Rất thấp                         |
| Phù hợp                      | Workload đa dạng, linh hoạt | Startup, team nhỏ, đơn giản      |

### Khi Nào Chọn Autopilot?

```
✅ Muốn giảm tối đa overhead vận hành node
✅ Workload có tải biến động nhiều (scale lên/xuống thường xuyên)
✅ Team nhỏ không có chuyên gia K8s platform
✅ Ưu tiên bảo mật (Autopilot có hardened node, không SSH)

❌ Cần chạy DaemonSet tùy chỉnh (log agent, monitoring agent)
❌ Cần privileged container hoặc hostPath
❌ Cần GPU node
❌ Cần tùy chỉnh kernel parameters
```

---

## Kiến Trúc GKE

### Regional Cluster vs Zonal Cluster

**Regional Cluster (Cluster Đa Vùng):** Control Plane và node được phân bổ qua nhiều zone — HA cao nhất.

```
Region: asia-southeast1 (Singapore)
├── Zone: asia-southeast1-a
│   ├── Control Plane Replica
│   └── Node Pool (2–3 nodes)
├── Zone: asia-southeast1-b
│   ├── Control Plane Replica
│   └── Node Pool (2–3 nodes)
└── Zone: asia-southeast1-c
    ├── Control Plane Replica
    └── Node Pool (2–3 nodes)
```

**Zonal Cluster (Cluster Đơn Vùng):** Control Plane trong 1 zone — đơn giản hơn, rẻ hơn nhưng không HA.

```
Zone: asia-southeast1-a
├── Control Plane (single zone)
└── Node Pool
```

### Control Plane trên GKE

Không giống EKS, GKE Control Plane chạy trong project ẩn của Google — bạn không thấy và không truy cập trực tiếp. Giao tiếp qua private endpoint được thiết lập tự động.

```bash
# Tạo regional cluster với private endpoint
gcloud container clusters create my-cluster \
  --region asia-southeast1 \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.0/28 \
  --enable-master-authorized-networks \
  --master-authorized-networks 10.0.0.0/8
```

---

## Node Pool — Nhóm Node

### Tạo và Quản Lý Node Pool

```bash
# Tạo cluster với node pool mặc định
gcloud container clusters create my-cluster \
  --region asia-southeast1 \
  --machine-type n2-standard-4 \
  --num-nodes 2 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10

# Thêm node pool đặc biệt (ví dụ: cho workload memory-intensive)
gcloud container node-pools create high-memory-pool \
  --cluster my-cluster \
  --region asia-southeast1 \
  --machine-type n2-highmem-8 \
  --num-nodes 1 \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 5 \
  --node-taints workload=memory-intensive:NoSchedule \
  --node-labels workload-type=memory-intensive
```

### Spot VM trong Node Pool (Tiết Kiệm Chi Phí)

```bash
gcloud container node-pools create spot-pool \
  --cluster my-cluster \
  --region asia-southeast1 \
  --spot \                              # Spot VM — tiết kiệm 60–91%
  --machine-type e2-standard-4 \
  --num-nodes 0 \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 20 \
  --node-taints cloud.google.com/gke-spot=true:NoSchedule
```

### Node Auto-provisioning (Cấp Phát Node Tự Động)

Tương tự Karpenter trên AWS — GKE tự động tạo node pool mới với machine type phù hợp khi có Pod pending.

```bash
gcloud container clusters update my-cluster \
  --enable-autoprovisioning \
  --max-cpu 1000 \
  --max-memory 1000 \
  --autoprovisioning-scopes=https://www.googleapis.com/auth/cloud-platform
```

---

## Workload Identity — Định Danh Workload

**Workload Identity** là cơ chế cho phép Kubernetes ServiceAccount (KSA) "giả mạo" Google Service Account (GSA) — truy cập GCP API mà không cần service account key file.

### Cơ Chế Hoạt Động

```
Pod
 │ (1) Gọi GCP API (ví dụ: GCS, BigQuery, Cloud SQL)
 ↓
GCP Client Library trong Pod
 │ (2) Đọc JWT token từ metadata server
 ↓
Metadata Server (chạy trên node, được GKE quản lý)
 │ (3) Xác thực KSA qua OIDC, kiểm tra IAM binding
 ↓
Google IAM
 │ (4) Trả về Access Token của GSA được liên kết
 ↓
Pod gọi GCP API thành công — không cần key file
```

### Cấu Hình Workload Identity

**Bước 1: Bật Workload Identity cho cluster**

```bash
# Bật khi tạo cluster
gcloud container clusters create my-cluster \
  --workload-pool=my-project.svc.id.goog

# Hoặc bật cho cluster đã tạo
gcloud container clusters update my-cluster \
  --workload-pool=my-project.svc.id.goog

# Bật trên từng node pool
gcloud container node-pools update default-pool \
  --cluster my-cluster \
  --workload-metadata=GKE_METADATA
```

**Bước 2: Tạo Google Service Account**

```bash
# Tạo GSA
gcloud iam service-accounts create my-app-gsa \
  --display-name "My App GSA"

# Gán role cho GSA (ví dụ: đọc GCS bucket)
gcloud projects add-iam-policy-binding my-project \
  --member "serviceAccount:my-app-gsa@my-project.iam.gserviceaccount.com" \
  --role roles/storage.objectViewer
```

**Bước 3: Tạo IAM binding giữa KSA và GSA**

```bash
# Cho phép KSA "giả mạo" GSA
gcloud iam service-accounts add-iam-policy-binding \
  my-app-gsa@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[my-namespace/my-ksa]"
```

**Bước 4: Annotate Kubernetes ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  namespace: my-namespace
  annotations:
    iam.gke.io/gcp-service-account: my-app-gsa@my-project.iam.gserviceaccount.com
```

**Bước 5: Pod dùng ServiceAccount đó**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
spec:
  serviceAccountName: my-ksa      # Pod tự động dùng Workload Identity
  containers:
    - name: app
      image: gcr.io/my-project/my-app:latest
      # GCP SDK tự detect Workload Identity — không cần cấu hình gì thêm
```

### Kiểm Tra Workload Identity Hoạt Động

```bash
# Chạy Pod test
kubectl run test-pod \
  --image google/cloud-sdk:slim \
  --serviceaccount my-ksa \
  --rm -it -- /bin/bash

# Trong Pod, kiểm tra identity hiện tại
gcloud auth list
# Phải thấy: my-app-gsa@my-project.iam.gserviceaccount.com

# Test truy cập GCS
gsutil ls gs://my-bucket/
```

---

## Release Channel — Kênh Phát Hành

GKE cho phép chọn tốc độ nhận bản cập nhật K8s qua Release Channel.

| Channel     | Phiên Bản K8s   | Mục Đích                                          |
| ----------- | --------------- | -------------------------------------------------- |
| **Rapid**   | Mới nhất        | Test tính năng mới, lab, staging                  |
| **Regular** | Ổn định         | Production thông thường (khuyến nghị)             |
| **Stable**  | Bảo thủ nhất    | Hệ thống yêu cầu ổn định tối đa, ít thay đổi     |
| **None**    | Tự kiểm soát    | Bạn tự quyết định thời điểm upgrade              |

```bash
# Tạo cluster với Regular channel
gcloud container clusters create my-cluster \
  --release-channel regular \
  --region asia-southeast1

# Chuyển sang kênh khác
gcloud container clusters update my-cluster \
  --release-channel stable
```

### Auto-upgrade Node

```bash
# Bật auto-upgrade với maintenance window (cửa sổ bảo trì)
gcloud container node-pools update default-pool \
  --cluster my-cluster \
  --enable-autoupgrade \
  --maintenance-window-start 2026-01-01T22:00:00Z \
  --maintenance-window-end 2026-01-01T06:00:00Z \
  --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SA,SU"
```

---

## Networking trên GKE

### VPC-native Cluster (Cluster Native VPC)

GKE hỗ trợ hai chế độ networking:

**Routes-based (cũ):** Pod IP được route qua bảng route của VPC.

**VPC-native / Alias IP (mới, khuyến nghị):** Pod nhận IP từ secondary IP range của subnet — native trong VPC, không cần custom route.

```bash
# Tạo VPC-native cluster (mặc định từ GKE 1.21+)
gcloud container clusters create my-cluster \
  --enable-ip-alias \
  --cluster-ipv4-cidr /16 \          # CIDR cho Pod IP range
  --services-ipv4-cidr /22            # CIDR cho Service IP range
```

### GKE Dataplane V2 (eBPF-based Networking)

GKE Dataplane V2 thay thế kube-proxy bằng eBPF (Extended Berkeley Packet Filter) — hiệu năng cao hơn, hỗ trợ Network Policy tốt hơn.

```bash
gcloud container clusters create my-cluster \
  --enable-dataplane-v2
```

### Private Cluster (Cluster Riêng Tư)

```bash
# Tạo private cluster — node không có external IP
gcloud container clusters create my-cluster \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.32/28
```

### Shared VPC (VPC Dùng Chung)

Nhiều GKE cluster trong các project khác nhau dùng chung một VPC — kiến trúc phổ biến trong tổ chức lớn.

---

## Load Balancer trên GKE

### Google Cloud Load Balancer (Cân Bằng Tải)

GKE tự động tạo các loại load balancer khi bạn tạo Service hoặc Ingress.

**Service LoadBalancer → External TCP/UDP Load Balancer:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    cloud.google.com/load-balancer-type: External   # Mặc định
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

**GKE Ingress → HTTP(S) Load Balancer (L7):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: gce               # Google Cloud HTTP(S) LB
    kubernetes.io/ingress.global-static-ip-name: my-static-ip
    networking.gke.io/managed-certificates: my-ssl-cert
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

**Internal Load Balancer (Cân Bằng Tải Nội Bộ):**

```yaml
metadata:
  annotations:
    cloud.google.com/load-balancer-type: Internal   # Chỉ trong VPC
```

### Gateway API (Thế Hệ Mới Của Ingress)

GKE hỗ trợ **Gateway API** — tiêu chuẩn mới của K8s community thay thế Ingress với khả năng cấu hình phong phú hơn.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: my-tls-secret
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
```

---

## Storage trên GKE

### Persistent Disk (Đĩa Lưu Trữ Bền Vững)

```yaml
# StorageClass dùng SSD Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd                      # pd-standard, pd-ssd, pd-extreme
  replication-type: regional-pd     # Replica qua 2 zone — HA hơn
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### Cloud Filestore (NFS Dùng Chung)

```yaml
# StorageClass dùng Filestore — hỗ trợ ReadWriteMany
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-rwx
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: STANDARD
  network: default
volumeBindingMode: Immediate
```

### Google Cloud Storage (GCS) với CSI Driver

```yaml
# Mount GCS bucket trực tiếp vào Pod
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gcs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  claimRef:
    namespace: default
    name: gcs-pvc
  csi:
    driver: gcsfuse.csi.storage.gke.io
    volumeHandle: my-gcs-bucket
    volumeAttributes:
      mountOptions: implicit-dirs
```

---

## Bảo Mật GKE

### Binary Authorization (Ủy Quyền Nhị Phân)

Chỉ cho phép deploy image đã được ký (signed) và phê duyệt.

```bash
# Bật Binary Authorization
gcloud container clusters update my-cluster \
  --enable-binauthz

# Tạo policy chỉ cho phép image từ Artifact Registry của project
gcloud container binauthz policy import policy.yaml
```

### GKE Security Posture (Tư Thế Bảo Mật GKE)

GKE tự động phân tích workload và cảnh báo các vấn đề bảo mật: privileged Pod, container chạy root, missing network policy...

```bash
# Bật Security Posture
gcloud container clusters update my-cluster \
  --enable-security-posture \
  --security-posture standard \
  --workload-vulnerability-scanning enterprise
```

### Shielded GKE Nodes (Node GKE Được Bảo Vệ)

```bash
gcloud container node-pools create secure-pool \
  --cluster my-cluster \
  --shielded-secure-boot \            # Xác thực boot image
  --shielded-integrity-monitoring \   # Theo dõi tính toàn vẹn
  --image-type cos_containerd         # Container-Optimized OS
```

### Private Google Access

Cho phép node không có public IP vẫn truy cập GCP APIs.

```bash
# Bật trên subnet
gcloud compute networks subnets update my-subnet \
  --region asia-southeast1 \
  --enable-private-ip-google-access
```

---

## Tích Hợp GCP Services

### Cloud SQL với Cloud SQL Auth Proxy

```yaml
# Sidecar container cho Cloud SQL Auth Proxy
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: DB_HOST
          value: "127.0.0.1"
        - name: DB_PORT
          value: "5432"
    - name: cloud-sql-proxy
      image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.11
      args:
        - "--structured-logs"
        - "--port=5432"
        - "my-project:asia-southeast1:my-db"    # Cloud SQL connection name
      securityContext:
        runAsNonRoot: true
```

### Artifact Registry (Registry Lưu Trữ Artifact)

```bash
# Cấu hình cluster pull image từ Artifact Registry
gcloud container clusters update my-cluster \
  --region asia-southeast1 \
  --update-addons=ConfigConnector=ENABLED

# Trong Autopilot/Standard với Workload Identity, không cần imagePullSecret
# Node tự dùng node service account để pull image
```

### Config Connector — Quản Lý GCP Resource từ K8s

Config Connector cho phép tạo GCP resource (bucket, database, topic) bằng K8s manifest.

```yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: my-bucket
  namespace: my-namespace
spec:
  location: asia-southeast1
  uniformBucketLevelAccess: true
  versioning:
    enabled: true
```

### Cloud Monitoring và Cloud Logging

GKE tích hợp sẵn với Google Cloud Monitoring và Logging — không cần cài thêm.

```bash
# Bật managed monitoring
gcloud container clusters update my-cluster \
  --enable-managed-prometheus     # Managed Prometheus Service
  --monitoring SYSTEM,WORKLOAD    # Thu metric system và workload
  --logging SYSTEM,WORKLOAD       # Thu log system và workload
```

---

## Tối Ưu Chi Phí GKE

### Committed Use Discount (Giảm Giá Cam Kết Sử Dụng)

```
1-year Committed Use:  tiết kiệm ~37%
3-year Committed Use:  tiết kiệm ~55%

Áp dụng cho: vCPU và Memory trên Compute Engine VM (node của GKE)
```

### Vertical Pod Autoscaler (VPA — Tự Động Điều Chỉnh Tài Nguyên Pod)

```bash
# Bật VPA
gcloud container clusters update my-cluster \
  --enable-vertical-pod-autoscaling
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto              # Tự động cập nhật resource request
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
```

### GKE Cost Optimization Best Practices

```bash
# 1. Bật cluster autoscaler với scale-to-zero
gcloud container node-pools update default-pool \
  --cluster my-cluster \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 10

# 2. Dùng Spot VM cho workload không quan trọng
gcloud container node-pools create batch-spot \
  --cluster my-cluster \
  --spot

# 3. Xem cost breakdown theo namespace (cần Cost Attribution)
gcloud container clusters update my-cluster \
  --resource-usage-bigquery-dataset my_dataset \
  --enable-network-egress-metering \
  --enable-resource-consumption-metering
```

---

## Vận Hành và Upgrade

### Kết Nối Cluster

```bash
# Lấy credentials
gcloud container clusters get-credentials my-cluster \
  --region asia-southeast1

# Kiểm tra kết nối
kubectl cluster-info
kubectl get nodes
```

### Upgrade Cluster

GKE với Release Channel sẽ tự động upgrade. Với channel None, bạn tự upgrade.

```bash
# Upgrade Control Plane
gcloud container clusters upgrade my-cluster \
  --master \
  --cluster-version 1.30.2-gke.1587000 \
  --region asia-southeast1

# Upgrade node pool (rolling update từng node)
gcloud container clusters upgrade my-cluster \
  --node-pool default-pool \
  --region asia-southeast1

# Xem phiên bản K8s có sẵn
gcloud container get-server-config \
  --region asia-southeast1
```

### Surge Upgrade (Nâng Cấp Theo Đợt)

```bash
# Cấu hình surge upgrade — tạo node mới trước khi xoá node cũ
gcloud container node-pools update default-pool \
  --cluster my-cluster \
  --max-surge-upgrade 1 \        # Tạo thêm 1 node mới khi upgrade
  --max-unavailable-upgrade 0    # Không cho phép node nào down khi upgrade
```

### Blue-Green Node Pool Upgrade

```bash
# Cách an toàn nhất: tạo node pool mới, migrate workload, xoá pool cũ

# Bước 1: Tạo node pool mới với phiên bản K8s mới
gcloud container node-pools create new-pool \
  --cluster my-cluster \
  --node-version 1.30.2-gke.1587000 \
  --machine-type n2-standard-4 \
  --num-nodes 3

# Bước 2: Cordon pool cũ (không schedule Pod mới vào)
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o name); do
  kubectl cordon $node
done

# Bước 3: Drain pool cũ (di chuyển Pod sang pool mới)
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o name); do
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data
done

# Bước 4: Xoá pool cũ
gcloud container node-pools delete default-pool \
  --cluster my-cluster \
  --region asia-southeast1
```

---

## Câu Hỏi Phỏng Vấn

**Q: GKE Autopilot và Standard khác nhau thế nào? Khi nào dùng mỗi loại?**

> Standard mode cho bạn toàn quyền kiểm soát node — chọn machine type, cấu hình taint/label, chạy DaemonSet tùy chỉnh. Autopilot thì GKE quản lý hoàn toàn node — bạn chỉ care workload, không quản lý node. Chọn Autopilot khi: muốn giảm overhead vận hành, workload serverless, tải biến động nhiều (trả tiền theo usage thực tế). Chọn Standard khi: cần DaemonSet tùy chỉnh (custom logging agent, security scanner), GPU, privileged container, hoặc cần tối ưu chi phí node thủ công.

**Q: Workload Identity hoạt động thế nào? So với EKS IRSA?**

> Workload Identity cho phép KSA (K8s ServiceAccount) "giả mạo" GSA (Google Service Account) qua OIDC federation. GKE tự cài metadata server trên mỗi node — khi Pod gọi GCP API, metadata server intercept, xác thực KSA qua OIDC, rồi trả về access token của GSA. Cơ chế tương tự IRSA nhưng đơn giản hơn: không cần tạo OIDC provider thủ công trên IAM (GKE tự lo), không cần trust policy phức tạp — chỉ cần IAM binding và annotation trên KSA.

**Q: GKE Release Channel là gì và nên chọn channel nào cho production?**

> Release Channel kiểm soát tốc độ GKE upgrade cluster. Rapid nhận bản K8s mới nhất nhưng ít kiểm thử nhất — chỉ dùng cho lab. Regular nhận bản đã qua kiểm thử vài tuần — phù hợp production thông thường. Stable nhận bản đã ổn định nhiều tháng — phù hợp hệ thống critical cần ổn định tối đa. Tôi thường chọn Regular cho production — cân bằng giữa tính năng mới và ổn định.

**Q: Làm thế nào để upgrade GKE cluster không downtime?**

> Với managed node pool: cấu hình `--max-surge-upgrade=1 --max-unavailable-upgrade=0` — GKE tạo node mới trước khi xoá node cũ, không có node bị down. Với Blue-Green approach: tạo node pool mới, cordon pool cũ, drain từng node (K8s reschedule Pod sang pool mới), rồi xoá pool cũ — an toàn nhất. Trước upgrade, cần test compatibility: đọc GKE release notes, check deprecated API, test trên staging cluster cùng version trước.

**Q: Config Connector là gì và khi nào dùng?**

> Config Connector là K8s add-on cho phép quản lý GCP resource (Cloud SQL, GCS bucket, Pub/Sub topic...) bằng K8s manifest và CRD. Khi dùng: team muốn quản lý infra hoàn toàn qua GitOps — cùng pipeline deploy K8s workload cũng deploy GCP resource. Lợi thế: drift detection (phát hiện sai lệch), state reconciliation tự động. Nhược điểm: học thêm CRD API, không phải tất cả GCP resource đều được hỗ trợ.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
