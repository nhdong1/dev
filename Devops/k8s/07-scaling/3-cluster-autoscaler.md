# Cluster Autoscaler — Tự Động Mở Rộng Node Cluster

> Hướng dẫn chi tiết về Cluster Autoscaler (CA — Tự Động Mở Rộng Node Cluster): cơ chế scale out khi Pod Pending, cơ chế scale in khi node nhàn rỗi, tích hợp với EKS/GKE/AKS, cấu hình node group, và các annotation kiểm soát hành vi scale.

## Mục Lục

1. [Cluster Autoscaler Là Gì?](#cluster-autoscaler-là-gì)
2. [Cơ Chế Scale Out](#cơ-chế-scale-out)
3. [Cơ Chế Scale In](#cơ-chế-scale-in)
4. [Cài Đặt Theo Cloud Provider](#cài-đặt-theo-cloud-provider)
5. [Cấu Hình CA Deployment](#cấu-hình-ca-deployment)
6. [Annotation Kiểm Soát Hành Vi](#annotation-kiểm-soát-hành-vi)
7. [Node Group và Priority](#node-group-và-priority)
8. [Debug và Giám Sát](#debug-và-giám-sát)
9. [Vấn Đề Thường Gặp](#vấn-đề-thường-gặp)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cluster Autoscaler Là Gì?

**Cluster Autoscaler (CA)** là component tự động điều chỉnh **số lượng node** trong cluster bằng cách:

- **Scale out (mở rộng):** Thêm node khi có Pod không thể được schedule do thiếu tài nguyên
- **Scale in (thu hẹp):** Xoá node khi node đó nhàn rỗi và Pod có thể chuyển sang node khác

CA tích hợp trực tiếp với API của cloud provider (AWS Auto Scaling Group, GCP Managed Instance Group, Azure VMSS) để thêm/xoá VM.

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  Pod Pending ────────────────────────────────────────┐  │
│  (không đủ tài nguyên)                               │  │
│                                                      │  │
│  ┌──────────────────┐   Phát hiện Pod Pending        │  │
│  │ Cluster          │ ←──────────────────────────────┘  │
│  │ Autoscaler       │                                    │
│  │                  │   Yêu cầu thêm VM                 │
│  └──────────────────┘ ──────────────────────────┐       │
│                                                 │       │
└─────────────────────────────────────────────────│───────┘
                                                  │
                    ┌─────────────────────────────▼───────┐
                    │        Cloud Provider API            │
                    │  AWS ASG / GCP MIG / Azure VMSS      │
                    │                                      │
                    │  Thêm VM mới vào node group          │
                    └─────────────────────────────────────┘
```

---

## Cơ Chế Scale Out

### Điều Kiện Scale Out

CA scale out khi:
1. Có Pod ở trạng thái `Pending`
2. Lý do Pending là **không đủ tài nguyên** (InsufficientCPU, InsufficientMemory, không đủ tài nguyên trên mọi node)
3. Có node group chưa đạt `maxSize`

### Quy Trình Scale Out

```
Mỗi 10 giây (sync period):

1. Quét tất cả Pod Pending
   → Lọc Pod Pending vì lý do tài nguyên

2. Mô phỏng: Pod này có chạy được trên node hiện tại không?
   → Không (không đủ CPU/memory/GPU...)

3. Mô phỏng: Nếu thêm 1 node mới vào node group, Pod có schedule được không?
   → Có → Trigger scale out

4. Gọi Cloud Provider API tăng node group size (ASG desired count ++)

5. VM mới boot, kubelet join cluster (~1–3 phút tuỳ cloud)

6. Scheduler đặt Pod Pending lên node mới
```

**Thời gian từ Pod Pending đến Pod Running:** thường 2–5 phút (phụ thuộc tốc độ VM boot của cloud provider).

---

## Cơ Chế Scale In

### Điều Kiện Scale In

CA kiểm tra scale in mỗi **10 giây** nhưng chỉ thực hiện khi:
1. Node có resource utilization < `--scale-down-utilization-threshold` (mặc định 50%)
2. Tất cả Pod trên node đó có thể được schedule trên node khác
3. Node đã ở trạng thái underutilized liên tục `--scale-down-unneeded-time` (mặc định 10 phút)

### Pod Ngăn Scale In

CA **không drain** node nếu có Pod:
- Được điều khiển bởi `kube-system` namespace (system DaemonSet)
- Có annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`
- Thuộc về một local PersistentVolume (không thể migrate)
- Vi phạm **PodDisruptionBudget (PDB)** nếu bị evict
- Là Pod tĩnh (static Pod) quản lý bởi kubelet

```
Node được đánh dấu "unneeded":
        │
        │  chờ --scale-down-unneeded-time (10 phút)
        ▼
CA kiểm tra lại điều kiện scale in
        │
        │  Không có Pod blocking?
        ▼
Drain node (evict tất cả Pod sang node khác)
        │
        │  chờ --scale-down-delay-after-delete (0s mặc định)
        ▼
Gọi Cloud Provider API xoá VM
        ▼
Node bị xoá khỏi cluster
```

---

## Cài Đặt Theo Cloud Provider

### Amazon EKS — IAM Permission Cho CA

```bash
# CA cần quyền đọc và điều chỉnh Auto Scaling Group
# Tạo IAM policy và gắn vào node IAM role hoặc dùng IRSA (IAM Roles for Service Accounts)
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": ["*"]
    }
  ]
}
```

### Tag Cho EKS Node Group (Auto Scaling Group)

```
# CA tìm ASG thuộc cluster thông qua tag này
k8s.io/cluster-autoscaler/<cluster-name>: owned
k8s.io/cluster-autoscaler/enabled: true
```

### Google GKE — Bật Node Auto Provisioning

```bash
# GKE có CA tích hợp sẵn — bật khi tạo cluster
gcloud container clusters create my-cluster \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=10 \
  --num-nodes=3

# Hoặc bật cho node pool hiện có
gcloud container node-pools update my-node-pool \
  --cluster=my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10
```

### Azure AKS — Bật Cluster Autoscaler

```bash
# Bật CA khi tạo AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# Bật CA cho cluster hiện có
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

---

## Cấu Hình CA Deployment

### Deploy CA Trên EKS (Self-Managed)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste           # chọn node group tốn ít tài nguyên nhất
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --balance-similar-node-groups    # cân bằng node giữa các node group tương tự
            - --scale-down-enabled=true
            - --scale-down-utilization-threshold=0.5
            - --scale-down-unneeded-time=10m
            - --scale-down-delay-after-add=10m
          resources:
            requests:
              cpu: 100m
              memory: 300Mi
            limits:
              cpu: 100m
              memory: 300Mi
```

### Expander — Chiến Lược Chọn Node Group

| Expander | Chiến Lược | Phù Hợp |
| -------- | ---------- | -------- |
| `least-waste` | Chọn node group để ít tài nguyên bị lãng phí nhất sau khi thêm Pod | Tối ưu chi phí |
| `most-pods` | Chọn node group có thể schedule được nhiều Pod Pending nhất | Tốc độ scale |
| `random` | Chọn ngẫu nhiên | Test/dev |
| `priority` | Dùng danh sách ưu tiên tuỳ chỉnh | Multi node group phức tạp |

---

## Annotation Kiểm Soát Hành Vi

### Ngăn Pod Bị Evict Khi Scale In

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-job
  annotations:
    # CA sẽ không evict Pod này khi drain node
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  containers:
    - name: critical-job
      image: my-critical-job:1.0
```

> **Lưu ý:** Dùng annotation này quá nhiều sẽ ngăn CA scale in bất kỳ node nào chứa Pod như vậy — làm tăng chi phí không cần thiết.

### Scale Down Node Cụ Thể Sớm Hơn

```yaml
# Nếu muốn CA scale down node này sớm hơn (dù chưa 10 phút)
# Hữu ích khi biết node sắp cần bảo trì
kubectl annotate node node-name \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=false
```

### Vô Hiệu Hoá Scale Down Cho Node Cụ Thể

```bash
# Giữ node này không bị CA xoá (VD: node có workload đặc biệt)
kubectl annotate node node-name \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

---

## Node Group và Priority

### Priority Expander — Ưu Tiên Node Group

```yaml
# ConfigMap định nghĩa thứ tự ưu tiên node group
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:                                         # priority thấp nhất
      - .*spot.*                                # Spot/Preemptible nodes
    50:
      - .*standard.*                            # Standard nodes
    100:                                        # priority cao nhất
      - .*memory-optimized.*                    # Memory-optimized nodes (đắt hơn)
```

> CA ưu tiên chọn node group có priority cao nhất trước. Điều này cho phép sử dụng **Spot Instance (máy ảo giá rẻ, có thể bị reclaim)** khi có thể, fallback sang On-Demand khi Spot không available.

### Mixed Instance Policy — Kết Hợp Spot và On-Demand

```
Thiết kế node group thực tế:
│
├── node-group-spot      (1–10 nodes, Spot, cho workload không critical)
│   priority: 10         ← CA chọn trước
│
├── node-group-ondemand  (2–5 nodes, On-Demand, luôn có mặt)
│   priority: 50         ← CA chọn khi Spot đầy hoặc không có
│
└── node-group-large     (0–3 nodes, On-Demand memory-optimized, đắt)
    priority: 5          ← CA chọn cuối cùng
```

---

## Debug và Giám Sát

### Xem CA Logs

```bash
# Xem log CA để hiểu tại sao scale out / không scale
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100

# Các message quan trọng cần tìm:
# "Scale-up: setting group ... to 5" → CA đang scale out
# "Scale-down: node ... is unneeded" → Node sắp bị xoá
# "pod ... can't be moved, blocking scale-down" → Pod đang ngăn scale in
# "No candidates for scale down" → Không node nào đủ điều kiện scale in
```

### Kiểm Tra Pod Pending Do Tài Nguyên

```bash
# Tìm Pod Pending
kubectl get pods -A --field-selector=status.phase=Pending

# Xem lý do Pod Pending
kubectl describe pod <pod-name> -n <namespace>
# Events:
#   Warning  FailedScheduling  0/3 nodes are available:
#            1 Insufficient cpu, 2 Insufficient memory.

# Xem CA đã phát hiện và phản ứng chưa
kubectl logs -n kube-system cluster-autoscaler-xxx | grep "Scale-up"
```

### Kiểm Tra Node Utilization

```bash
# Xem resource usage của từng node
kubectl top nodes

# Xem tài nguyên allocatable trên mỗi node
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Output:
#   Resource           Requests     Limits
#   --------           --------     ------
#   cpu                1850m (92%)  2600m (130%)   ← gần đầy!
#   memory             2358Mi (73%) 3358Mi (104%)
```

---

## Vấn Đề Thường Gặp

### 1. CA Không Scale Out Dù Có Pod Pending

**Nguyên nhân có thể:**
- Pod Pending vì lý do khác (không phải tài nguyên): NodeSelector không khớp, Taint không có Toleration, PVC không bind được
- Node group đã đạt `maxSize`
- CA không có permission gọi cloud provider API

```bash
# Kiểm tra lý do Pod Pending
kubectl describe pod pending-pod-name | grep -A 10 "Events:"

# Kiểm tra CA có permission không
kubectl logs -n kube-system cluster-autoscaler-xxx | grep -i "error\|forbidden"

# Kiểm tra node group đã max chưa (EKS)
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names my-asg \
  --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity}'
```

### 2. CA Không Scale In Dù Node Nhàn Rỗi

**Nguyên nhân phổ biến:**
- Có Pod có annotation `safe-to-evict: "false"`
- DaemonSet Pod không thể evict
- PodDisruptionBudget ngăn evict
- Node đã có local storage (emptyDir, hostPath)

```bash
# Tìm Pod ngăn scale in trên node
kubectl describe node node-name | grep -A 20 "Non-terminated Pods:"

# Kiểm tra PDB
kubectl get pdb -A

# Xem CA đánh giá node như thế nào
kubectl logs -n kube-system cluster-autoscaler-xxx | grep "node-name"
```

### 3. Scale Out Chậm (Pod Pending Nhiều Phút)

**Nguyên nhân:** VM boot time chậm, hoặc image pull lâu trên node mới.

**Giải pháp:** Sử dụng **node overprovisioning** — luôn giữ một số node "dự phòng" không có workload thực, để Pod có thể schedule ngay lập tức thay vì chờ VM mới.

```yaml
# Overprovisioning: Deployment dùng PriorityClass thấp
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 2       # 2 Pod "giả" giữ chỗ trên 2 node dự phòng
  template:
    spec:
      priorityClassName: overprovisioning   # priority thấp — bị evict ngay khi Pod thật cần
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
```

---

## Câu Hỏi Phỏng Vấn

**Cluster Autoscaler scale out khi nào? Cơ chế ra sao?**

> CA scale out khi phát hiện Pod ở trạng thái `Pending` do không đủ tài nguyên trên bất kỳ node hiện có nào. CA **mô phỏng** — không chờ thật — liệu thêm một node vào node group có giải quyết được Pod Pending không. Nếu có, CA gọi Cloud Provider API (AWS Auto Scaling Group, GCP MIG, Azure VMSS) để tăng node count. VM mới join cluster sau 1–3 phút (tuỳ cloud), sau đó Scheduler đặt Pod lên node mới. Quan trọng: CA scale out dựa trên `resource.requests`, không phải resource usage thực tế — vì thế request phải chính xác để CA tính toán đúng.

**Cluster Autoscaler scale in khi nào? Tại sao scale in rủi ro hơn scale out?**

> CA scale in khi node có tổng resource utilization < 50% (dựa trên request, không phải usage thực tế) trong 10 phút liên tiếp, VÀ tất cả Pod trên đó có thể chạy được trên node khác. Scale in rủi ro hơn vì: (1) Pod bị evict — nếu ứng dụng không handle graceful shutdown tốt có thể gây request drop; (2) PodDisruptionBudget có thể bị vi phạm nếu cấu hình sai; (3) Evict StatefulSet Pod có thể gây mất session; (4) Scale in chậm hơn scale out — mặc định chờ 10 phút trước khi drain. Để an toàn: luôn có `PodDisruptionBudget` và `minReplicas >= 2` cho critical service.

**Khác nhau giữa Cluster Autoscaler và Karpenter?**

> **Cluster Autoscaler** là component cổ điển, hoạt động với **Node Group** (ASG, MIG, VMSS) được định nghĩa trước — chỉ scale in/out các loại VM đã cấu hình trong node group. **Karpenter** (AWS, open source) là thế hệ mới — thay vì node group, Karpenter nhìn trực tiếp vào Pod spec (CPU, memory, instance-type selector, Spot preference) và **provision VM tối ưu nhất** trực tiếp từ cloud API. Karpenter nhanh hơn (scale out trong ~60s vs 2–3 phút), linh hoạt hơn (không bị giới hạn instance type cố định), và tiết kiệm chi phí hơn (chọn instance phù hợp nhất cho workload). Hiện tại Karpenter chủ yếu trên AWS; CA vẫn là lựa chọn phổ biến trên GKE và AKS.
