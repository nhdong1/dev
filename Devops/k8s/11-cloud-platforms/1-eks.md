# Amazon EKS — Elastic Kubernetes Service

> Hướng dẫn toàn diện về Amazon EKS (Elastic Kubernetes Service — Dịch Vụ Kubernetes Đàn Hồi): kiến trúc, IRSA, Fargate, node group, add-on và vận hành production trên AWS.

## Mục Lục

1. [Tổng Quan EKS](#tổng-quan-eks)
2. [Kiến Trúc EKS](#kiến-trúc-eks)
3. [Node Group — Nhóm Node](#node-group--nhóm-node)
4. [IRSA — IAM Roles for Service Accounts](#irsa--iam-roles-for-service-accounts)
5. [EKS Pod Identity](#eks-pod-identity)
6. [Fargate Profile](#fargate-profile)
7. [EKS Add-on](#eks-add-on)
8. [Networking trên EKS](#networking-trên-eks)
9. [Load Balancer trên EKS](#load-balancer-trên-eks)
10. [Storage trên EKS](#storage-trên-eks)
11. [Bảo Mật EKS](#bảo-mật-eks)
12. [Cluster Autoscaler và Karpenter](#cluster-autoscaler-và-karpenter)
13. [Vận Hành và Upgrade](#vận-hành-và-upgrade)
14. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan EKS

Amazon EKS là managed Kubernetes service trên AWS — ra mắt năm 2018. AWS quản lý toàn bộ Control Plane (mặt điều khiển), bạn quản lý Worker Node và workload.

### Chi Phí Cơ Bản

```
Control Plane:  $0.10/giờ/cluster (~$73/tháng)
Worker Node:    Chi phí EC2 instance thông thường
Fargate:        Tính theo vCPU và memory thực tế dùng
Data Transfer:  Có thể tốn kém — lưu ý cross-AZ traffic
```

### Các Thành Phần Chính

```
EKS Cluster
├── Control Plane (AWS quản lý)
│   ├── API Server (chạy trên AWS managed infrastructure)
│   ├── etcd (multi-AZ, AWS quản lý backup)
│   ├── Scheduler
│   └── Controller Manager
│
├── Node Group (Bạn quản lý)
│   ├── Managed Node Group — AWS tự upgrade
│   ├── Self-Managed Node Group — Bạn tự quản lý AMI
│   └── Fargate Profile — Serverless, không có node
│
└── Add-on (Tiện ích)
    ├── VPC CNI — networking
    ├── CoreDNS — DNS cluster
    ├── kube-proxy — routing
    ├── EBS CSI Driver — block storage
    └── EFS CSI Driver — shared file storage
```

---

## Kiến Trúc EKS

### Control Plane

EKS Control Plane chạy trong VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) của AWS, được tách biệt hoàn toàn với VPC của khách hàng. Giao tiếp qua private endpoint.

```
AWS Account của bạn              AWS Account của EKS (ẩn)
┌─────────────────────┐          ┌─────────────────────────┐
│  VPC của bạn        │          │  EKS Control Plane       │
│                     │  ←ENI→  │  Multi-AZ                │
│  Worker Node 1      │          │  API Server              │
│  Worker Node 2      │          │  etcd (managed backup)   │
│  Worker Node 3      │          │  Scheduler               │
│                     │          │  Controller Manager      │
└─────────────────────┘          └─────────────────────────┘
```

**ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi):** AWS tạo ENI trong subnet của bạn để Worker Node kết nối với Control Plane.

### Endpoint Access Modes (Chế Độ Truy Cập Endpoint)

```yaml
# Public Endpoint — kubectl có thể gọi từ Internet
# (Không khuyến nghị cho production)
resourcesVpcConfig:
  endpointPublicAccess: true
  endpointPrivateAccess: false

# Private Endpoint — chỉ truy cập trong VPC
# (Khuyến nghị cho production)
resourcesVpcConfig:
  endpointPublicAccess: false
  endpointPrivateAccess: true

# Cả hai — public có whitelist IP, private cho node
resourcesVpcConfig:
  endpointPublicAccess: true
  publicAccessCidrs:
    - "203.0.113.0/24"    # IP office
  endpointPrivateAccess: true
```

---

## Node Group — Nhóm Node

### Managed Node Group (Nhóm Node Được Quản Lý)

AWS tự động quản lý lifecycle của EC2 instance: cung cấp AMI chuẩn, thực hiện rolling update khi upgrade.

```yaml
# eksctl tạo managed node group
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-southeast-1

managedNodeGroups:
  - name: ng-production
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 50
    ssh:
      allow: false                  # Tắt SSH truy cập trực tiếp
    labels:
      role: worker
    tags:
      Environment: production
    iam:
      withAddonPolicies:
        autoScaler: true            # Policy cho Cluster Autoscaler
        albIngress: true            # Policy cho AWS Load Balancer Controller
        ebs: true                   # Policy cho EBS CSI Driver
```

### Self-Managed Node Group (Nhóm Node Tự Quản Lý)

Bạn tự cung cấp EC2 instance và AMI. Linh hoạt hơn nhưng phức tạp hơn.

```bash
# Tạo launch template với custom AMI và userdata
# UserData phải chứa lệnh join cluster:
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --b64-cluster-ca $CLUSTER_CA \
  --apiserver-endpoint $API_ENDPOINT
```

### Spot Instance trong Node Group

```yaml
managedNodeGroups:
  - name: ng-spot
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m4.xlarge
    spot: true
    minSize: 0
    maxSize: 20
    labels:
      lifecycle: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule          # Chỉ Pod có toleration mới lên đây
```

```yaml
# Pod tolerate Spot Node
tolerations:
  - key: spot
    operator: Equal
    value: "true"
    effect: NoSchedule
```

---

## IRSA — IAM Roles for Service Accounts

**IRSA (IAM Roles for Service Accounts — Vai Trò IAM cho ServiceAccount)** cho phép Pod truy cập AWS API bằng IAM Role tạm thời — không cần access key.

### Cơ Chế Hoạt Động

```
Pod
 │ (1) Đọc JWT token từ projected volume
 ↓
AWS SDK trong Pod
 │ (2) Gửi JWT token đến AWS STS
 ↓
AWS STS (Security Token Service — Dịch Vụ Mã Thông Báo Bảo Mật)
 │ (3) Xác thực token qua OIDC Provider của EKS cluster
 │ (4) Kiểm tra điều kiện trong IAM Role Trust Policy
 ↓
AWS STS trả về Temporary Credentials (tối đa 12 giờ)
 │
 ↓
Pod gọi AWS API với Temporary Credentials (S3, DynamoDB, SQS...)
```

### Cấu Hình IRSA

**Bước 1: Tạo OIDC Provider cho EKS Cluster**

```bash
# Lấy OIDC URL của cluster
OIDC_URL=$(aws eks describe-cluster \
  --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text)

# Tạo OIDC Identity Provider trên AWS IAM
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

**Bước 2: Tạo IAM Role với Trust Policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub":
            "system:serviceaccount:my-namespace:my-service-account",
          "oidc.eks.ap-southeast-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud":
            "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

**Bước 3: Annotate ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: my-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-irsa-role
```

**Bước 4: Pod tự động nhận Token**

```yaml
# Pod spec — không cần thêm gì, token được inject tự động
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
spec:
  serviceAccountName: my-service-account    # Gắn SA đã có annotation
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: AWS_DEFAULT_REGION
          value: ap-southeast-1
      # AWS SDK sẽ tự dùng token từ /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

**Cách Tạo Nhanh với eksctl**

```bash
eksctl create iamserviceaccount \
  --name my-service-account \
  --namespace my-namespace \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts
```

---

## EKS Pod Identity

**EKS Pod Identity** là thế hệ mới của IRSA — ra mắt 2023, đơn giản hoá đáng kể cấu hình.

### So Sánh IRSA vs Pod Identity

| Tiêu Chí                      | IRSA                        | Pod Identity                |
| ----------------------------- | --------------------------- | --------------------------- |
| Cấu hình OIDC Provider        | Bắt buộc tạo thủ công       | Không cần                   |
| Trust Policy phức tạp         | Có (chứa OIDC condition)    | Không có (association riêng)|
| Hỗ trợ cross-account          | Có                          | Không (trong cùng account)  |
| Cú pháp cấu hình              | Phức tạp hơn                | Đơn giản hơn                |
| Hỗ trợ phiên bản K8s          | Tất cả                      | Từ K8s 1.24+                |

### Cấu Hình Pod Identity

```bash
# Bước 1: Cài EKS Pod Identity add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent

# Bước 2: Tạo IAM Role (trust policy đơn giản hơn)
# Trust principal: pods.eks.amazonaws.com

# Bước 3: Tạo Pod Identity Association — không cần annotate ServiceAccount
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace my-namespace \
  --service-account my-service-account \
  --role-arn arn:aws:iam::123456789:role/my-role
```

---

## Fargate Profile

**Fargate** cho phép chạy Pod serverless — AWS tự cấp phát compute, bạn không quản lý node.

### Cách Hoạt Động

```
Bạn định nghĩa Fargate Profile với namespace và label selector
         ↓
Khi Pod match selector được schedule
         ↓
AWS tự động cấp phát micro-VM riêng cho Pod
         ↓
Pod chạy trong isolated environment — không chia sẻ node với Pod khác
```

### Cấu Hình Fargate Profile

```yaml
# eksctl config
fargateProfiles:
  - name: fp-default
    selectors:
      - namespace: default
      - namespace: kube-system
  - name: fp-batch
    selectors:
      - namespace: batch
        labels:
          workload-type: batch
```

```bash
# Hoặc tạo bằng AWS CLI
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name fp-batch \
  --pod-execution-role-arn arn:aws:iam::123456789:role/FargatePodExecutionRole \
  --selectors namespace=batch,labels={workload-type=batch} \
  --subnets subnet-private-1 subnet-private-2
```

### Giới Hạn Của Fargate

```
❌ Không hỗ trợ DaemonSet
❌ Không hỗ trợ hostPath volume
❌ Không hỗ trợ privileged container
❌ Không hỗ trợ GPU
❌ Tài nguyên tối đa: 4 vCPU, 30 GB memory
❌ Không hỗ trợ chạy trên public subnet
✅ Hỗ trợ IRSA đầy đủ
✅ Tự động patch OS và runtime
✅ Tính phí chính xác theo usage
```

---

## EKS Add-on

**Add-on** là các component quản lý bởi AWS — được cập nhật và vá lỗi tự động.

### Các Add-on Phổ Biến

| Add-on                   | Chức Năng                                              |
| ------------------------ | ------------------------------------------------------ |
| `vpc-cni`                | AWS VPC CNI — cấp IP VPC trực tiếp cho Pod            |
| `coredns`                | DNS resolution trong cluster                           |
| `kube-proxy`             | Network rules trên mỗi node                            |
| `aws-ebs-csi-driver`     | Provisioning EBS volume (Block Storage — Lưu Trữ Khối)|
| `aws-efs-csi-driver`     | Provisioning EFS volume (Shared File Storage)          |
| `eks-pod-identity-agent` | Agent hỗ trợ Pod Identity                             |

### Quản Lý Add-on

```bash
# Liệt kê add-on hiện tại
aws eks list-addons --cluster-name my-cluster

# Xem phiên bản sẵn có
aws eks describe-addon-versions --addon-name vpc-cni

# Cập nhật add-on lên phiên bản mới
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.1-eksbuild.1

# Với eksctl
eksctl get addons --cluster my-cluster
eksctl update addon --name vpc-cni --cluster my-cluster
```

---

## Networking trên EKS

### AWS VPC CNI

Mặc định EKS dùng AWS VPC CNI — mỗi Pod nhận một IP thực trong VPC subnet.

```
VPC CIDR: 10.0.0.0/16
├── Subnet Private 1 (AZ-a): 10.0.1.0/24
│   ├── Node 1 IP: 10.0.1.10
│   ├── Pod 1A IP: 10.0.1.100      ← IP thực trong VPC
│   └── Pod 1B IP: 10.0.1.101      ← IP thực trong VPC
└── Subnet Private 2 (AZ-b): 10.0.2.0/24
    ├── Node 2 IP: 10.0.2.10
    └── Pod 2A IP: 10.0.2.100      ← IP thực trong VPC
```

**Giới hạn IP:** Mỗi EC2 instance type có giới hạn số ENI và IP — ảnh hưởng đến số Pod tối đa trên mỗi node.

```bash
# Xem giới hạn Pod trên node
kubectl describe node <node-name> | grep "pods\|Allocatable"

# Tính: Max Pod = (số ENI × IPs/ENI) - 1
# Ví dụ m5.xlarge: 3 ENI × 15 IP = 45 Pod tối đa
```

### Prefix Delegation (Ủy Quyền Tiền Tố)

Tăng số IP per ENI bằng cách gán /28 prefix block thay vì từng IP đơn lẻ.

```bash
# Bật prefix delegation cho VPC CNI
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true

# Một /28 prefix = 16 IP → tăng đáng kể số Pod tối đa
```

---

## Load Balancer trên EKS

### AWS Load Balancer Controller

**AWS Load Balancer Controller** (thay thế AWS ALB Ingress Controller) tạo ALB (Application Load Balancer) và NLB (Network Load Balancer) từ K8s resource.

```bash
# Cài đặt qua Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Ingress tạo ALB:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip         # IP mode — Pod IP trực tiếp
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/group.name: shared-alb   # Nhiều Ingress dùng chung 1 ALB
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

**Service tạo NLB:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
```

---

## Storage trên EKS

### EBS CSI Driver (Elastic Block Store — Lưu Trữ Khối Đàn Hồi)

```yaml
# StorageClass dùng EBS gp3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer   # Tạo volume đúng AZ của Pod
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain
```

```yaml
# PVC dùng EBS
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce             # EBS chỉ mount được trên 1 node
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 20Gi
```

### EFS CSI Driver (Elastic File System — Hệ Thống File Đàn Hồi)

EFS hỗ trợ ReadWriteMany — nhiều Pod trên nhiều node cùng đọc/ghi.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap           # Dùng EFS Access Point
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
```

---

## Bảo Mật EKS

### aws-auth ConfigMap — Ánh Xạ IAM với K8s RBAC

```yaml
# kubectl edit configmap aws-auth -n kube-system
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789:role/DevOpsTeamRole
      username: devops-user
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/alice
      username: alice
      groups:
        - developers
```

### EKS Access Entry (Thay Thế aws-auth ConfigMap)

Từ EKS 1.29+, AWS giới thiệu Access Entry API — quản lý quyền truy cập qua AWS API thay vì ConfigMap.

```bash
# Tạo access entry cho IAM role
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/DevRole \
  --type STANDARD

# Gán quyền
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=development
```

### Bảo Mật Secrets với AWS Secrets Manager

```yaml
# Dùng Secrets Store CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "my-db-password"
        objectType: "secretsmanager"
        objectAlias: "db-password"
```

---

## Cluster Autoscaler và Karpenter

### Cluster Autoscaler

```yaml
# Cluster Autoscaler deployment
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
            - --cloud-provider=aws
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --balance-similar-node-groups       # Phân bổ đều giữa các AZ
            - --skip-nodes-with-local-storage=false
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
```

### Karpenter — Tự Động Cấp Phát Node Thế Hệ Mới

**Karpenter** (do AWS phát triển) thay thế Cluster Autoscaler — phản ứng nhanh hơn và linh hoạt hơn.

```yaml
# NodePool — định nghĩa loại node Karpenter có thể tạo
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000                         # Giới hạn tổng CPU của pool
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized   # Thu nhỏ khi node rảnh
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest             # Amazon Linux 2023 AMI mới nhất
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
```

**So sánh Cluster Autoscaler vs Karpenter:**

| Tiêu Chí                   | Cluster Autoscaler         | Karpenter                    |
| -------------------------- | -------------------------- | ---------------------------- |
| Thời gian scale-up         | 3–5 phút                   | 30–60 giây                   |
| Chọn instance type         | Từ danh sách cố định       | Tự động chọn phù hợp nhất    |
| Consolidation              | Hạn chế                    | WhenEmpty / WhenUnderutilized|
| Quản lý                    | Phức tạp hơn               | Đơn giản hơn                 |

---

## Vận Hành và Upgrade

### Kiểm Tra Trạng Thái Cluster

```bash
# Xem thông tin cluster
aws eks describe-cluster --name my-cluster

# Xem tất cả node
kubectl get nodes -o wide

# Xem add-on
aws eks list-addons --cluster-name my-cluster

# Xem Fargate profile
aws eks list-fargate-profiles --cluster-name my-cluster
```

### Upgrade Cluster

```bash
# Bước 1: Upgrade Control Plane (từng phiên bản một)
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.30

# Chờ Control Plane upgrade xong
aws eks wait cluster-active --name my-cluster

# Bước 2: Upgrade add-on
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.1-eksbuild.1

# Bước 3: Upgrade Managed Node Group
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name ng-production
```

### Backup và Restore với Velero

```bash
# Cài Velero với S3 backend
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-velero-backup-bucket \
  --backup-location-config region=ap-southeast-1 \
  --use-node-agent

# Tạo backup
velero backup create my-backup --include-namespaces production

# Restore
velero restore create --from-backup my-backup
```

---

## Câu Hỏi Phỏng Vấn

**Q: EKS khác self-managed K8s ở điểm nào quan trọng nhất?**

> Điểm khác biệt lớn nhất là AWS quản lý toàn bộ Control Plane với SLA 99.9% — bạn không cần lo etcd backup, Control Plane HA hay upgrade. Quan trọng hơn, EKS tích hợp native với AWS ecosystem: IRSA cho phép Pod truy cập AWS API không cần credential, VPC CNI cho Pod nhận IP VPC thực, ALB Controller tạo Load Balancer tự động. Chi phí thêm $0.10/giờ cho managed Control Plane thường đáng khi so với chi phí vận hành.

**Q: Giải thích IRSA và tại sao nó an toàn hơn dùng access key?**

> IRSA dùng OIDC federation: EKS API Server phát hành JWT token cho mỗi ServiceAccount, AWS STS xác thực token đó và cấp temporary credential (tối đa 12 giờ). Lý do an toàn hơn: (1) Không có static secret lưu trong cluster hay environment variable — không thể bị lộ. (2) Credential tự expire — không cần rotate thủ công. (3) Mỗi ServiceAccount có Role riêng — least privilege dễ thực thi. (4) AWS CloudTrail log đầy đủ theo từng ServiceAccount — audit trail rõ ràng.

**Q: Khi nào dùng Fargate thay vì EC2 node?**

> Dùng Fargate khi: (1) Workload batch job, data processing không cần chạy liên tục — trả tiền đúng theo usage, tiết kiệm hơn EC2 trả phí dù idle. (2) Muốn giảm hoàn toàn overhead vận hành node — không cần patch OS, không cần quản lý AMI. (3) Yêu cầu bảo mật cao — mỗi Pod chạy trong micro-VM riêng biệt. Không dùng Fargate cho: workload cần DaemonSet, cần GPU, cần mount hostPath, hoặc workload cần nhiều hơn 4 vCPU / 30 GB memory.

**Q: Karpenter khác Cluster Autoscaler thế nào?**

> Cluster Autoscaler scale node group đã định sẵn — chậm (3–5 phút) vì phải chờ ASG (Auto Scaling Group) cung cấp EC2 mới. Karpenter trực tiếp gọi EC2 API để tạo node đúng loại phù hợp với Pod đang Pending — nhanh hơn (30–60 giây) và chọn instance type tối ưu về chi phí. Karpenter cũng có consolidation — tự gộp Pod về ít node hơn khi tải giảm, tiết kiệm chi phí hơn.

**Q: Làm thế nào để migrate workload từ public node sang private node?**

> (1) Tạo node group mới trong private subnet. (2) Taint node group cũ với `NoSchedule` để Pod mới không schedule vào. (3) Drain từng node cũ bằng `kubectl drain --ignore-daemonsets` — K8s sẽ reschedule Pod lên node mới trong private subnet. (4) Verify Pod chạy đúng trên node mới. (5) Xoá node group cũ. Đảm bảo trước khi migrate: Load Balancer health check path vẫn hoạt động, Security Group cho phép traffic từ ALB vào private node.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
