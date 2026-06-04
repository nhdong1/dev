# EKS Node Groups — Managed Nodes, Self-managed Nodes & Fargate Profiles

> **Node Groups (Nhóm Node)** xác định nơi Pod thực sự chạy trong EKS cluster. Bạn có 3 lựa chọn: Managed Node Groups (AWS quản lý lifecycle), Self-managed Nodes (bạn tự quản lý), và Fargate Profiles (serverless — không cần nghĩ đến nodes). Mỗi lựa chọn có trade-offs về độ phức tạp, chi phí và tính linh hoạt.

---

## 📚 Mục Lục

1. [Ba Loại Data Plane](#ba-loại-data-plane)
2. [Managed Node Groups — Nhóm Node Được Quản Lý](#managed-node-groups--nhóm-node-được-quản-lý)
3. [Self-managed Node Groups — Nhóm Node Tự Quản Lý](#self-managed-node-groups--nhóm-node-tự-quản-lý)
4. [Fargate Profiles — Hồ Sơ Fargate Không Máy Chủ](#fargate-profiles--hồ-sơ-fargate-không-máy-chủ)
5. [Node Group Autoscaling — Tự Động Co Giãn Nhóm Node](#node-group-autoscaling)
6. [Karpenter — Autoscaler Thế Hệ Mới](#karpenter--autoscaler-thế-hệ-mới)
7. [Taint & Toleration — Kiểm Soát Nơi Pod Chạy](#taint--toleration)
8. [So Sánh Tổng Hợp](#so-sánh-tổng-hợp)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Ba Loại Data Plane

```
┌────────────────────────────────────────────────────────────────────┐
│                        EKS Data Plane Options                      │
│                                                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │   Managed Node  │  │  Self-managed   │  │    Fargate       │   │
│  │     Groups      │  │  Node Groups    │  │   Profiles       │   │
│  │                 │  │                 │  │                  │   │
│  │ EC2 instances   │  │ EC2 instances   │  │ Serverless       │   │
│  │ AWS manages     │  │ You manage      │  │ micro-VMs        │   │
│  │ kubelet, AMI    │  │ everything      │  │ AWS manages all  │   │
│  │                 │  │                 │  │                  │   │
│  │ Phổ biến nhất   │  │ Khi cần custom  │  │ Khi không muốn  │   │
│  │ cho production  │  │ AMI, kernel     │  │ quản lý nodes    │   │
│  └─────────────────┘  └─────────────────┘  └──────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

---

## Managed Node Groups — Nhóm Node Được Quản Lý

### Đặc Điểm

- AWS tự động cung cấp, cập nhật và xóa EC2 instances
- Dùng **EKS-optimized AMI (Amazon Machine Image — Ảnh Máy Ảo Tối Ưu EKS)** — pre-configured kubelet, containerd
- Hỗ trợ **rolling update** khi upgrade node version
- Tích hợp với **EC2 Auto Scaling Groups** tự động
- Hỗ trợ **Spot instances** và **On-Demand instances**

### Tạo Managed Node Group

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --node-role arn:aws:iam::123456789:role/EKSNodeRole \
  --subnets subnet-abc123 subnet-def456 subnet-ghi789 \
  --instance-types t3.medium t3.large \
  --ami-type AL2_x86_64 \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --disk-size 50 \
  --labels role=application,env=production \
  --taints key=dedicated,value=production,effect=NoSchedule
```

### EKS-Optimized AMI Types

| AMI Type | Mô Tả | Khi Dùng |
|---|---|---|
| `AL2_x86_64` | Amazon Linux 2, x86_64 | Mặc định, hầu hết workloads |
| `AL2_x86_64_GPU` | Amazon Linux 2 + NVIDIA drivers | GPU workloads (ML/AI) |
| `AL2_ARM_64` | Amazon Linux 2, ARM/Graviton | Graviton instances (tiết kiệm ~20% chi phí) |
| `AL2023_x86_64_STANDARD` | Amazon Linux 2023 | Thế hệ mới, recommended |
| `BOTTLEROCKET_x86_64` | Bottlerocket OS | Security-focused, immutable OS |
| `WINDOWS_*` | Windows Server | Windows containers |

### Node Group IAM Role — Vai Trò IAM Cho Node Group

```json
// Policy tối thiểu cần cho managed node group
{
  "policies": [
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  ]
}
```

> **Lưu ý quan trọng:** IAM role của node là identity cho kubelet/kube-proxy. Đừng thêm các quyền S3, DynamoDB vào node role — dùng IRSA thay thế để cấp quyền cho individual Pods.

### Update Strategy — Chiến Lược Cập Nhật

```
Rolling Update (mặc định):
  1. Cordon node (đánh dấu unschedulable)
  2. Drain node (evict tất cả Pods có thể evict)
  3. Terminate EC2 instance
  4. Launch EC2 instance mới với AMI mới
  5. Join cluster
  6. Lặp lại cho đến hết tất cả nodes

Cấu hình:
  maxUnavailable: 1    # Số node tối đa có thể down cùng lúc
  maxSurge: 1          # Số node extra có thể tạo tạm thời khi update
```

---

## Self-managed Node Groups — Nhóm Node Tự Quản Lý

### Khi Nào Dùng Self-managed

- Cần **custom AMI** với kernel modules đặc biệt, security agents
- Cần instance types chưa được managed node group hỗ trợ
- Cần **custom bootstrap scripts** phức tạp
- Cần cấu hình kubelet flags nâng cao
- Dùng **Spot Instances với diversification (đa dạng hóa)** phức tạp

### Setup Self-managed Node Group

```bash
# 1. Tạo Launch Template với EKS bootstrap script
cat <<EOF > user-data.sh
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=role=batch,lifecycle=spot' \
  --b64-cluster-ca <BASE64_CA> \
  --apiserver-endpoint https://<API_ENDPOINT>
EOF

# 2. Tạo Auto Scaling Group trỏ đến EKS cluster
# 3. Tag ASG với kubernetes.io/cluster/<cluster-name>=owned
```

### Nhược Điểm Self-managed

- Bạn phải tự upgrade AMI khi có security patches
- Phải tự implement rolling update logic
- Quản lý Auto Scaling Group manually
- Không có tích hợp sẵn với EKS upgrade workflow

---

## Fargate Profiles — Hồ Sơ Fargate Không Máy Chủ

### Fargate Profile Hoạt Động Thế Nào

```
Kubernetes Scheduler                Fargate Profile Selector
    │                                       │
    │ Pod created in namespace "backend"     │
    │ với label app=api                      │
    │──────────────────────────────────────>│
    │                                       │ Match? Yes!
    │                                       │ Namespace: backend
    │                                       │ Labels: app=api
    │                                       │
    │                           AWS creates micro-VM riêng cho Pod
    │                           (dedicated, isolated kernel)
    │                           Pod nhận IP từ VPC subnet
    │ Pod Running                            │
    │<──────────────────────────────────────│
```

**Mỗi Pod Fargate = 1 VM riêng biệt** — isolation mạnh, không share kernel với Pod khác.

### Tạo Fargate Profile

```bash
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name backend-profile \
  --pod-execution-role-arn arn:aws:iam::123456789:role/EKSFargatePodExecutionRole \
  --subnets subnet-abc123 subnet-def456 \
  --selectors \
    'namespace=backend' \
    'namespace=frontend,labels={tier=web}'
```

### Fargate Profile Selectors — Bộ Chọn Hồ Sơ Fargate

```yaml
# Selector theo namespace
selectors:
  - namespace: production

# Selector theo namespace + labels
selectors:
  - namespace: backend
    labels:
      app: api
      tier: web

# Quan trọng: Pod PHẢI match ít nhất 1 selector trong profile
# Nếu không match → Pod bị Pending vô thời hạn (không có node nào để chạy)
```

### Fargate IAM Pod Execution Role — Vai Trò Thực Thi Pod

```json
// Policy cần thiết cho Fargate Pod Execution Role
{
  "policies": [
    "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  ]
}
// Cho phép Fargate pull ECR images và ghi CloudWatch Logs
```

### Giới Hạn Của Fargate

| Giới Hạn | Chi Tiết |
|---|---|
| **Không hỗ trợ DaemonSets** | Không chạy được Node monitoring agents như Datadog Node Agent |
| **Không hỗ trợ hostNetwork** | Pod không thể dùng network của host |
| **Không hỗ trợ privileged containers** | Không thể mount host paths, không có root access |
| **Không hỗ trợ hostPort** | Không expose port trực tiếp trên node |
| **EBS volumes không hỗ trợ** | Chỉ dùng EFS cho persistent storage |
| **Giới hạn CPU/Memory** | Max 4 vCPU và 30 GB RAM per Pod |
| **Stateful workloads khó** | StatefulSets cần cẩn thận |
| **Cold start chậm hơn** | VM khởi động lâu hơn container trên EC2 node |

### Khi Nào Dùng Fargate vs Managed Node Groups

```
Dùng Fargate khi:
  ✅ Workload burst không đoán trước — pay per Pod, không trả tiền idle nodes
  ✅ Muốn zero node management overhead
  ✅ Isolation mạnh giữa Pods (security requirement)
  ✅ Batch jobs chạy tạm thời

Dùng Managed Node Groups khi:
  ✅ Cần DaemonSets (monitoring agents, log agents)
  ✅ Workload cần GPU
  ✅ Cần EBS persistent volumes
  ✅ Muốn tối ưu chi phí với Spot Instances
  ✅ Workload liên tục, dễ dự đoán tài nguyên
```

---

## Node Group Autoscaling

### Cluster Autoscaler (CA) — Bộ Tự Động Co Giãn Cluster

```
Hoạt động:
  Scale Out (Mở Rộng): Pod Pending → CA tìm Node Group phù hợp → tăng desiredSize
  Scale In (Thu Hẹp): Node idle > 10 phút (không có Pod không thể di chuyển) → giảm desiredSize

Cấu hình quan trọng:
  --scale-down-utilization-threshold=0.5  # Node dưới 50% CPU/Memory là idle
  --scale-down-delay-after-add=10m        # Chờ 10 phút sau scale-out mới xem xét scale-in
  --expander=least-waste                  # Chọn node group tốn ít resource nhất

Annotation quan trọng để block scale-in:
  cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
  # Đặt trên Pod để CA không evict và không xóa node chứa Pod này
```

```yaml
# Ví dụ: Cluster Autoscaler deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

### HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)

```
HPA scale số lượng Pod, Cluster Autoscaler scale số lượng Nodes.
Hai cơ chế phối hợp nhau:

  High load
    → HPA thêm Pod
    → Pod Pending (không đủ node)
    → Cluster Autoscaler thêm node
    → Pod được schedule và chạy

  Low load
    → HPA giảm Pod
    → Node idle
    → Cluster Autoscaler xóa node
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Karpenter — Autoscaler Thế Hệ Mới

### Karpenter vs Cluster Autoscaler

| Tính Năng | Cluster Autoscaler | Karpenter |
|---|---|---|
| **Tốc độ scale** | Chậm hơn (phải chờ ASG) | Nhanh hơn (launch EC2 trực tiếp) |
| **Chọn instance type** | Giới hạn bởi node group config | Tự động chọn best-fit instance |
| **Bin packing** | Kém hơn | Tốt hơn — tối ưu số node |
| **Spot diversity** | Cần cấu hình nhiều node groups | Tự động đa dạng hóa |
| **Cấu hình** | Phức tạp với nhiều node groups | Đơn giản hơn với NodePool |
| **AWS Native** | Có | Có (được AWS phát triển) |

### Karpenter NodePool Concept

```yaml
# Thay vì tạo nhiều Node Groups, Karpenter dùng NodePool
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]   # Thử Spot trước, fallback On-Demand
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["c", "m", "r"]          # Compute, Memory, R-series
      - key: karpenter.k8s.aws/instance-size
        operator: NotIn
        values: ["nano", "micro", "small"]  # Tránh instance quá nhỏ
  limits:
    cpu: 1000           # Tối đa 1000 CPU cores trong pool này
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s   # Xóa node idle sau 30 giây
```

---

## Taint & Toleration

### Khái Niệm

- **Taint (Vết Bẩn):** Đặt trên node để **từ chối** Pod không có toleration tương ứng
- **Toleration (Sự Chịu Đựng):** Đặt trên Pod để **chấp nhận** schedule lên node có taint đó

### Ví Dụ Thực Tế

```bash
# Taint node để chỉ dành cho ML workloads
kubectl taint nodes gpu-node-1 dedicated=ml-workload:NoSchedule

# Pod KHÔNG có toleration → bị từ chối schedule lên gpu-node-1
# Pod CÓ toleration sau → được phép schedule:
```

```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ml-workload"
    effect: "NoSchedule"
  containers:
  - name: tensorflow
    image: tensorflow/tensorflow:latest-gpu
```

### Taint Effects

| Effect | Hành Vi |
|---|---|
| `NoSchedule` | Pod mới không được schedule lên node. Pod đang chạy không bị ảnh hưởng |
| `PreferNoSchedule` | Scheduler cố gắng tránh, nhưng không bắt buộc |
| `NoExecute` | Pod mới không được schedule VÀ Pod đang chạy bị evict nếu không có toleration |

### Node Affinity — Ưu Ái Node (Nâng Cao Hơn Taint)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # Bắt buộc
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]  # Chỉ chạy ở 2 AZ này
      preferredDuringSchedulingIgnoredDuringExecution:  # Ưu tiên nhưng không bắt buộc
      - weight: 100
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["m5.2xlarge"]
```

---

## So Sánh Tổng Hợp

| Tiêu Chí | Managed Node Group | Self-managed | Fargate |
|---|---|---|---|
| **Độ phức tạp** | Thấp | Cao | Thấp nhất |
| **Kiểm soát** | Trung bình | Cao nhất | Thấp nhất |
| **Chi phí** | EC2 giá chuẩn | EC2 giá chuẩn | Đắt hơn EC2 |
| **AMI updates** | AWS tự động | Bạn tự làm | Không cần |
| **DaemonSets** | ✅ | ✅ | ❌ |
| **GPU** | ✅ | ✅ | ❌ |
| **EBS volumes** | ✅ | ✅ | ❌ |
| **Spot support** | ✅ | ✅ | ❌ |
| **Startup time** | 2-3 phút | 2-3 phút | Nhanh nhưng có overhead |
| **Phù hợp** | Hầu hết production | Custom requirements | Batch, burst workloads |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Managed Node Group vs Fargate — khi nào chọn cái nào?

**Trả lời:**
> Managed Node Groups phù hợp cho hầu hết production workloads: hỗ trợ DaemonSets (cần cho monitoring agents), GPU, EBS volumes, Spot instances. Fargate phù hợp khi muốn zero node management — nhất là batch jobs, burst workloads không đoán trước, hoặc khi security yêu cầu isolation mạnh giữa Pods. Giới hạn của Fargate: không DaemonSets, không EBS, không privileged containers, max 4vCPU/30GB per Pod.

### Câu 2: Cluster Autoscaler và HPA phối hợp nhau thế nào?

**Trả lời:**
> HPA (Horizontal Pod Autoscaler) scale số Pod dựa trên CPU/memory/custom metrics. Cluster Autoscaler scale số Nodes. Khi HPA thêm Pod nhưng cluster thiếu tài nguyên, Pod rơi vào Pending. Cluster Autoscaler phát hiện Pod Pending, xác định Node Group phù hợp, tăng desiredSize → EC2 mới launch → Pod được schedule. Ngược lại khi tải giảm: HPA giảm Pod → Node idle → Cluster Autoscaler drain và terminate node.

### Câu 3: Taint và Toleration dùng để làm gì? Cho ví dụ thực tế.

**Trả lời:**
> Taint đặt trên node để ngăn Pod không phù hợp schedule vào đó. Toleration trên Pod cho phép Pod vượt qua taint. Ví dụ thực tế: Đánh taint `dedicated=gpu:NoSchedule` trên GPU nodes. Chỉ ML training Pods có toleration tương ứng mới được schedule lên đó. Các Pods thông thường không bị ảnh hưởng nhưng sẽ không tranh GPU nodes với ML workloads — đảm bảo GPU nodes không bị "chiếm" bởi workloads không cần GPU.

### Câu 4: Karpenter ưu việt hơn Cluster Autoscaler ở điểm nào?

**Trả lời:**
> Karpenter launch EC2 instances trực tiếp (không qua ASG) → nhanh hơn đáng kể (~30 giây vs 2-3 phút). Karpenter tự động chọn instance type tối ưu (bin packing tốt hơn) → tiết kiệm chi phí. Với Spot, Karpenter tự động diversify qua nhiều instance types mà không cần cấu hình nhiều node groups. Tuy nhiên Karpenter phức tạp hơn để setup và debug ban đầu.

---

**Tiếp theo:** [3-eks-networking.md](./3-eks-networking.md) — VPC CNI, CoreDNS, Ingress, và Network Policies.
