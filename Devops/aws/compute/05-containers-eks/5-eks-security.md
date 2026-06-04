# EKS Security — RBAC, IRSA, Pod Security & Secrets Encryption

> Bảo mật EKS là bảo mật nhiều lớp: từ IAM authentication (xác thực), RBAC authorization (ủy quyền), Pod security (bảo mật Pod), đến network segmentation (phân đoạn mạng) và secrets management (quản lý bí mật). File này đi sâu vào từng lớp với ví dụ thực tế.

---

## 📚 Mục Lục

1. [Authentication vs Authorization — Xác Thực vs Ủy Quyền](#authentication-vs-authorization)
2. [RBAC — Role-Based Access Control](#rbac--role-based-access-control)
3. [IRSA — IAM Roles for Service Accounts](#irsa--iam-roles-for-service-accounts)
4. [Pod Security Standards — Tiêu Chuẩn Bảo Mật Pod](#pod-security-standards)
5. [Secrets Management — Quản Lý Bí Mật](#secrets-management)
6. [Image Security — Bảo Mật Image](#image-security)
7. [Security Checklist Production](#security-checklist-production)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Authentication vs Authorization

### Luồng Xác Thực Và Ủy Quyền

```
Developer / Application
    │
    │ kubectl / API call
    ▼
API Server
    │
    ├─ 1. AUTHENTICATION (Xác Thực): Bạn là ai?
    │      - IAM User/Role → aws-iam-authenticator
    │      - Service Account Token → Kubernetes built-in
    │      - OIDC (OpenID Connect) provider
    │
    ├─ 2. AUTHORIZATION (Ủy Quyền): Bạn được làm gì?
    │      - RBAC rules (Roles, ClusterRoles, Bindings)
    │
    ├─ 3. ADMISSION CONTROL (Kiểm Soát Đầu Vào): Request có hợp lệ không?
    │      - Mutating Webhooks (sửa đổi request)
    │      - Validating Webhooks (kiểm tra request)
    │      - Pod Security Admission
    │
    └─ 4. ETCD: Lưu object nếu qua hết các bước trên
```

### EKS Authentication — Xác Thực IAM

```bash
# EKS dùng aws-iam-authenticator để xác thực
# kubectl gọi: aws eks get-token --cluster-name my-cluster
# Token được decode bởi API Server qua authenticator webhook
# AWS IAM identity được map sang Kubernetes identity qua aws-auth ConfigMap

# Xem aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml
```

```yaml
# aws-auth ConfigMap — map IAM Users/Roles sang Kubernetes users/groups
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
    - rolearn: arn:aws:iam::123456789:role/DevOpsTeam
      username: devops-user
      groups:
      - devops-group     # Phải tạo RBAC RoleBinding cho group này
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/alice
      username: alice
      groups:
      - system:masters   # Toàn quyền cluster-admin (cẩn thận!)
```

> **Lưu ý:** Từ EKS 1.28, AWS khuyến nghị dùng **EKS Access Entries API** thay vì chỉnh sửa aws-auth ConfigMap thủ công — an toàn hơn và ít lỗi người dùng hơn.

---

## RBAC — Role-Based Access Control

### Các Object RBAC

```
Role: Quyền trong 1 namespace cụ thể
ClusterRole: Quyền trên toàn cluster (hoặc non-namespaced resources)

RoleBinding: Gán Role/ClusterRole cho User/Group/ServiceAccount trong 1 namespace
ClusterRoleBinding: Gán ClusterRole cho User/Group/ServiceAccount trên toàn cluster
```

### Ví Dụ: Role Cho Developer

```yaml
# Role: Cho phép đọc/xem trong namespace "development"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]        # Cho phép xem logs
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods/exec"]       # Cho phép exec vào Pod (cẩn thận!)
  verbs: ["create"]

---
# RoleBinding: Gán role cho group "dev-team"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: development
subjects:
- kind: Group
  name: dev-team          # Phải match với group trong aws-auth
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Ví Dụ: ClusterRole Cho CI/CD Pipeline

```yaml
# ClusterRole cho CI/CD — deploy lên mọi namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cicd-deployer-binding
subjects:
- kind: ServiceAccount
  name: cicd-service-account
  namespace: ci-system
roleRef:
  kind: ClusterRole
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Kiểm Tra RBAC

```bash
# Kiểm tra user/group có quyền gì
kubectl auth can-i get pods --namespace production
kubectl auth can-i get pods --namespace production --as alice
kubectl auth can-i get pods --namespace production --as-group dev-team

# Liệt kê tất cả RoleBindings trong namespace
kubectl get rolebindings -n production -o wide

# Xem chi tiết quyền của một Role
kubectl describe role developer-readonly -n development
```

---

## IRSA — IAM Roles for Service Accounts

### Vấn Đề IRSA Giải Quyết

```
Cách CŨ (Instance Profile — Hồ Sơ Máy Chủ):
  Tất cả Pods trên 1 node → dùng chung IAM role của node
  Pod A cần đọc S3-Bucket-A
  Pod B cần ghi DynamoDB-Table-B
  → Phải cấp CÙNG role có cả S3 VÀ DynamoDB cho node
  → Vi phạm least privilege (đặc quyền tối thiểu)
  → Nếu Pod B bị compromise → attacker cũng có quyền S3

Cách MỚI (IRSA — IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts):
  Pod A có ServiceAccount "s3-reader" → IAM Role chỉ đọc S3-Bucket-A
  Pod B có ServiceAccount "dynamo-writer" → IAM Role chỉ ghi DynamoDB-Table-B
  → Mỗi Pod có quyền riêng
  → Compromise Pod B không ảnh hưởng S3
```

### Cơ Chế IRSA

```
1. EKS cluster có OIDC Provider (OpenID Connect — Nhà Cung Cấp Xác Thực)
   URL: https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

2. IAM Role có trust policy tin tưởng OIDC Provider:
   "Ai có token từ OIDC provider này, ServiceAccount 'my-sa' trong namespace 'my-ns'
    thì được assume role này"

3. EKS inject OIDC token vào Pod qua projected volume
4. AWS SDK trong Pod dùng token này để lấy temporary credentials từ STS
   (STS — Security Token Service — Dịch Vụ Token Bảo Mật)
```

### Thiết Lập IRSA Step-by-Step

```bash
# Bước 1: Lấy OIDC provider URL của cluster
aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text
# Output: https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLEID

# Bước 2: Tạo OIDC Identity Provider trong IAM
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Bước 3: Tạo IAM Role với trust policy
OIDC_PROVIDER=$(aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's/https:\/\///')

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/${OIDC_PROVIDER}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
        "${OIDC_PROVIDER}:sub": "system:serviceaccount:production:s3-reader-sa"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name EKS-S3-Reader \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name EKS-S3-Reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```yaml
# Bước 4: Tạo ServiceAccount với annotation trỏ đến IAM Role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/EKS-S3-Reader

---
# Bước 5: Pod dùng ServiceAccount đó
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: s3-reader-sa    # Gắn IRSA ServiceAccount
  containers:
  - name: app
    image: my-app
    # AWS SDK tự động dùng OIDC token → get temp credentials → access S3
    # Không cần hardcode access key/secret trong code!
```

### Kiểm Tra IRSA Hoạt Động

```bash
# Exec vào Pod và kiểm tra identity
kubectl exec -it my-pod -- aws sts get-caller-identity
# Output phải show: arn:aws:sts::123456789:assumed-role/EKS-S3-Reader/...

# Kiểm tra environment variables được inject
kubectl exec -it my-pod -- env | grep AWS
# AWS_ROLE_ARN=arn:aws:iam::123456789:role/EKS-S3-Reader
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

---

## Pod Security Standards — Tiêu Chuẩn Bảo Mật Pod

### Ba Mức Độ Bảo Mật

```
Privileged (Đặc Quyền): Không hạn chế — dùng cho system components
Baseline (Cơ Bản): Ngăn các privilege escalation phổ biến nhất
Restricted (Hạn Chế): Best practice security — yêu cầu chặt chẽ nhất
```

### Bật Pod Security Admission Cho Namespace

```yaml
# Đặt label trên namespace để enforce Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce: Pod không tuân thủ sẽ bị từ chối
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    # audit: Log cảnh báo nhưng không từ chối
    pod-security.kubernetes.io/audit: restricted
    # warn: Hiển thị cảnh báo nhưng không từ chối
    pod-security.kubernetes.io/warn: restricted
```

### Pod Security Context — Ngữ Cảnh Bảo Mật Pod

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true          # Không chạy với root user
    runAsUser: 1000             # UID cụ thể
    runAsGroup: 3000
    fsGroup: 2000               # Group cho volume files
    seccompProfile:             # Seccomp profile (giới hạn syscalls)
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:v1.2.3        # Tag cụ thể, không dùng "latest"
    securityContext:
      allowPrivilegeEscalation: false   # Không cho leo thang đặc quyền
      readOnlyRootFilesystem: true      # Filesystem read-only
      capabilities:
        drop:
        - ALL                           # Bỏ tất cả Linux capabilities
        add:
        - NET_BIND_SERVICE              # Chỉ thêm lại nếu thực sự cần
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"          # Luôn đặt memory limit để tránh OOM
        cpu: "500m"
```

---

## Secrets Management — Quản Lý Bí Mật

### Kubernetes Secrets — Hạn Chế

```
Mặc định Kubernetes Secret chỉ được base64 encoded, KHÔNG mã hóa!
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
→ Hiển thị password thẳng ra

Cần 2 lớp bảo vệ:
  1. Encryption at Rest: Mã hóa etcd bằng KMS (AWS Key Management Service)
  2. RBAC: Giới hạn ai được đọc Secrets
```

### Encryption at Rest với KMS — Mã Hóa Khi Lưu

```bash
# Bật Secrets encryption khi tạo cluster
aws eks create-cluster \
  --name my-cluster \
  --encryption-config '[{
    "provider": {
      "keyArn": "arn:aws:kms:us-east-1:123:key/abc-def"
    },
    "resources": ["secrets"]
  }]'
  # Secrets trong etcd được mã hóa bằng KMS key của bạn
  # Ngay cả AWS cũng không đọc được nếu không có KMS key
```

### AWS Secrets Manager Integration — Tích Hợp AWS Secrets Manager

```yaml
# Dùng AWS Secrets and Configuration Provider (ASCP)
# Secrets Manager → CSI Driver → Mount vào Pod như file

apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "production/db-credentials"
        objectType: "secretsmanager"
        jmesPath:
        - path: "username"
          objectAlias: "db-username"
        - path: "password"
          objectAlias: "db-password"
  secretObjects:                    # Đồng thời tạo Kubernetes Secret
  - secretName: db-credentials-k8s
    type: Opaque
    data:
    - objectName: db-username
      key: username
    - objectName: db-password
      key: password

---
# Pod mount secrets từ Secrets Manager
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: app-sa     # Cần IRSA có quyền đọc Secrets Manager
  containers:
  - name: app
    image: my-app
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials-k8s    # Từ Kubernetes Secret được sync
          key: username
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets
      readOnly: true
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets
```

### External Secrets Operator — Giải Pháp Phổ Biến Khác

```yaml
# External Secrets Operator (ESO) — sync secrets từ AWS vào Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h            # Sync lại mỗi giờ (bắt rotation)
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials          # Kubernetes Secret được tạo
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: production/db-credentials   # AWS Secrets Manager path
      property: password
  - secretKey: username
    remoteRef:
      key: production/db-credentials
      property: username
```

---

## Image Security — Bảo Mật Image

### Best Practices Container Image

```yaml
# 1. Dùng image từ ECR (Elastic Container Registry) private, không Docker Hub
# 2. Scan image trước khi deploy (Amazon Inspector, Trivy, Snyk)
# 3. Dùng image digest (không phải tag) để pin version chính xác

containers:
- name: app
  # BAD: Dùng latest — không biết version nào đang chạy
  image: my-app:latest

  # GOOD: Dùng immutable digest
  image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app@sha256:abc123...

# 4. Distroless hoặc minimal base image
  image: gcr.io/distroless/java17-debian11   # Không có shell, không có package manager
```

### ECR Image Scanning — Quét Lỗ Hổng Image

```bash
# Bật scan tự động khi push image
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Xem kết quả scan
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.2.3

# Bật Enhanced Scanning (Amazon Inspector) cho ECR
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"repositoryFilters": [{"filter":"*","filterType":"WILDCARD"}],
             "scanFrequency":"CONTINUOUS_SCAN"}]'
```

### OPA Gatekeeper — Policy Enforcement

```yaml
# Enforce: Chỉ cho phép image từ ECR của công ty
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allow-only-ecr
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production", "staging"]
  parameters:
    repos:
    - "123456789.dkr.ecr.us-east-1.amazonaws.com"
    - "123456789.dkr.ecr.ap-southeast-1.amazonaws.com"
# Pod dùng image từ Docker Hub → bị từ chối
```

---

## Security Checklist Production

### IAM & Authentication

```
✅ EKS cluster dùng private endpoint (hoặc public+private với CIDR restriction)
✅ aws-auth ConfigMap (hoặc EKS Access Entries) cấu hình đúng
✅ Không có IAM user với system:masters (dùng role thay vì user)
✅ IRSA cho mọi workload cần AWS API access — không dùng instance profile
✅ Rotate IAM credentials và audit CloudTrail thường xuyên
```

### RBAC

```
✅ Không dùng ClusterAdmin cho developer thông thường
✅ Mỗi application có ServiceAccount riêng (không dùng default)
✅ Principle of least privilege: chỉ cấp quyền cần thiết
✅ Audit RBAC định kỳ: ai có quyền gì
✅ Namespace isolation: dev/staging/production tách biệt
```

### Pod Security

```
✅ Pod Security Standards: Restricted cho production namespaces
✅ Không chạy container với root user
✅ readOnlyRootFilesystem: true (khi có thể)
✅ Resource requests và limits đặt cho mọi container
✅ allowPrivilegeEscalation: false
✅ Drop ALL capabilities, chỉ add lại khi cần
```

### Secrets

```
✅ Secrets encryption at rest với KMS
✅ Dùng AWS Secrets Manager hoặc External Secrets Operator
✅ Không hardcode secrets trong YAML hoặc environment variables
✅ RBAC cho Secrets: giới hạn ai đọc được
✅ Rotate secrets định kỳ
```

### Networking & Images

```
✅ Network Policies: default deny, whitelist traffic cần thiết
✅ Ingress với TLS termination (HTTPS)
✅ Chỉ dùng image từ ECR private (không Docker Hub)
✅ Image scanning (Amazon Inspector) trên mọi image
✅ Không dùng image tag "latest" — dùng digest hoặc immutable tags
✅ Control plane logging bật (API, Audit, Authenticator)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: IRSA hoạt động thế nào? Tại sao nó tốt hơn instance profile?

**Trả lời:**
> IRSA dùng OIDC federation: EKS cluster có OIDC provider, mỗi ServiceAccount được gán IAM Role qua annotation. Khi Pod start, EKS inject OIDC token vào Pod. AWS SDK dùng token này để exchange lấy temporary STS credentials cho IAM Role tương ứng. Ưu điểm so với instance profile: (1) Mỗi Pod có IAM role riêng thay vì chia sẻ role của node — least privilege ở cấp Pod; (2) Credentials tự động rotate — không cần quản lý; (3) Audit trail rõ ràng trong CloudTrail — biết Pod nào gọi AWS API nào; (4) Compromise 1 Pod không ảnh hưởng Pods khác trên cùng node.

### Câu 2: Làm thế nào để ngăn developer đọc production secrets?

**Trả lời:**
> Kết hợp nhiều lớp: (1) Namespace separation — dev không có quyền vào namespace production; (2) RBAC — Role trong namespace production không có verbs get/list trên Secrets resource; (3) Encryption at rest với KMS — dù ai access etcd trực tiếp cũng không đọc được; (4) Dùng AWS Secrets Manager với IRSA — chỉ ServiceAccount được authorize mới lấy được secret; (5) Audit logging — CloudTrail và Kubernetes audit log ghi lại mọi lần ai access secret.

### Câu 3: Pod Security Standards là gì? Khi nào dùng Restricted?

**Trả lời:**
> Pod Security Standards có 3 mức: Privileged (không hạn chế), Baseline (ngăn privilege escalation phổ biến), Restricted (best practice — yêu cầu chặt chẽ nhất: non-root, read-only filesystem, drop all capabilities, seccomp). Enforcement qua labels trên namespace. Restricted phù hợp cho production workloads; system namespaces (kube-system) cần Privileged vì components như kube-proxy cần elevated privileges. Migrate lên Restricted từng bước — bắt đầu với warn/audit mode để phát hiện violations trước khi enforce.

### Câu 4: Mô tả kiến trúc bảo mật EKS end-to-end cho production app.

**Trả lời:**
> (1) **Network**: Private endpoint cho API Server, Pods ở private subnet, Security Groups chặt chẽ cho nodes, Network Policies cho Pod-to-Pod isolation; (2) **IAM/Auth**: Developer access qua IAM Role → aws-auth mapping → RBAC, không ai có cluster-admin trừ break-glass account; (3) **Workload**: IRSA cho mọi app cần AWS API, Pod Security Restricted cho production namespace, resource limits trên mọi container; (4) **Secrets**: KMS encryption for etcd, External Secrets Operator sync từ Secrets Manager, không có secrets trong YAML; (5) **Images**: Chỉ ECR private, Enhanced Scanning liên tục, OPA Gatekeeper enforce repo whitelist; (6) **Observability**: CloudTrail cho audit, Kubernetes audit logs, GuardDuty Runtime Monitoring.

---

## 📌 Tóm Tắt Module 05-containers-eks

| Chủ Đề | File | Điểm Cốt Lõi |
|---|---|---|
| Architecture | 1-eks-architecture.md | Control plane (AWS managed), data plane (bạn managed), add-ons |
| Node Groups | 2-node-groups.md | Managed NG, Self-managed, Fargate — trade-offs |
| Networking | 3-eks-networking.md | VPC CNI (Pod IP = VPC IP), CoreDNS, ALB Ingress |
| Storage | 4-eks-storage.md | EBS (RWO, high perf), EFS (RWX, shared), StatefulSets |
| Security | 5-eks-security.md | RBAC, IRSA (per-Pod IAM), Pod Security, Secrets encryption |

**EKS vs ECS quyết định cuối cùng:**
> Nếu team không có Kubernetes expertise và workload không yêu cầu K8s ecosystem → Chọn ECS Fargate. Nếu đang dùng K8s, cần multi-cloud portability, hoặc workload phức tạp → Chọn EKS.
